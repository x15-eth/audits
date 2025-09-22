## Telcoin
[Contest details](https://cantina.xyz/code/26d5255b-6f68-46cf-be55-81dd565d9d16/overview)

### [High-01] QuorumWaiter rejection logic inverts byzantine fault tolerance

**Description**

In `quorum_waiter.rs`, the rejection handler flips the sign (subtracting rejected stake instead of adding it) so the protocol’s BFT guarantees fall apart.

```rust
Err(WaiterError::Rejected(stake)) => {  
    rejected_stake -= stake;  // @audit-> Should be += stake  
    available_stake -= stake;  
}
```

The `rejected_stake` variable is initialized to 0.

```rust
// Stake from a committee member that has rejected this batch.
 let mut rejected_stake = 0;
 ```

When validators reject a batch, the current logic subtracts their stake, making `rejected_stake` negative. This breaks the critical safety check that determines when a batch should be permanently rejected

```rust

if rejected_stake > max_rejected_stake {  
    // Can no longer reach quorum because our batch was explicitly rejected by  
    // to much stack.  
    break Err(QuorumWaiterError::QuorumRejected);  
}
```

Since `max_rejected_stake` is calculated as a positive value, negative `rejected_stake` values will never trigger this condition. As a result malicious validators can reject batches without consequence, as their rejections effectively become approvals in the quorum calculation

Any validator in the network can trigger this bug simply by rejecting a batch during normal consensus operations. No special privileges, capital requirements, or complex coordination is needed.

**Recommendation**

Change the rejection handling logic to correctly accumulate rejected stake:

```rust
Err(WaiterError::Rejected(stake)) => {  
    rejected_stake += stake;  // FIXED: Add instead of subtract  
    available_stake -= stake;  
}
```

### [High-02] Delegation parameter swap breaks fund recovery and authorization

**Description**

A critical parameter swap bug in the `delegateStake()` function completely breaks the delegation mechanism in the Telcoin Network ConsensusRegistry.

When creating delegation records, the validator and delegator addresses are stored in reversed positions within the Delegation struct, causing systematic failures in fund recovery, reward claiming, and access control.

The bug occurs in `ConsensusRegistry.sol` where the Delegation struct is instantiated with swapped parameters:
```solidity
// BUGGY CODE - Parameters are swapped
delegations[validatorAddress] = Delegation(
    blsPubkeyHash, 
    msg.sender,        // delegator address stored in validatorAddress field
    validatorAddress,  // validator address stored in delegator field  
    validatorVersion, 
    nonce + 1
);
```
Expected struct definition (from `IStakeManager.sol`):
```solidity
struct Delegation {
    bytes32 blsPubkeyHash;
    address validatorAddress;  // Should contain validator address
    address delegator;         // Should contain delegator address
    uint8 validatorVersion;
    uint64 nonce;
}
```
As a result. the `_getRecipient()` function returns `delegation.delegator`, which incorrectly contains the validator address instead of the delegator address:
```solidity
function _getRecipient(address validatorAddress) internal view returns (address) {
    Delegation storage delegation = delegations[validatorAddress];
    address recipient = delegation.delegator;  // Returns validator address instead of delegator
    if (recipient == address(0x0)) recipient = validatorAddress;
    return recipient;
}
```
Functions like `unstake()` and `claimStakeRewards()` perform authorization checks that fail for legitimate delegators:
```solidity
function unstake(address validatorAddress) external override whenNotPaused nonReentrant {
    // ... other checks ...
    address recipient = _getRecipient(validatorAddress);  // Returns wrong address
    if (msg.sender != validatorAddress && msg.sender != recipient) revert NotRecipient(recipient);
    // Delegator gets NotRecipient error because recipient is validator address
}
```
Also when stakes and rewards are distributed, they are sent to the wrong recipient address, potentially causing:

- Delegators unable to recover their staked funds
- Validators receiving funds intended for delegators
- Complete breakdown of the delegation trust model

While EIP-712 signature verification uses correct parameter ordering in `delegationDigest()`, the stored delegation data is corrupted, creating a mismatch between signed intentions and stored state.

**Recommendation**

Correct the parameter order in the Delegation struct instantiation

```solidity
// FIXED CODE - Correct parameter order
delegations[validatorAddress] = Delegation(
    blsPubkeyHash,
    validatorAddress,  // Correct: validator address in validatorAddress field
    msg.sender,        // Correct: delegator address in delegator field
    validatorVersion,
    nonce + 1
);

```

### [Low-01] `WorkerNetwork` event loop causes infinite CPU spin on channel closure

**Description**

In `WorkerNetwork::spawn`, once the network events channel closes there’s no exit path out of the nested loops. There is no shutdown flag or break so the code just spins forever, pegging one CPU core at 100%.
```rust
pub fn spawn(mut self, epoch_task_spawner: &TaskSpawner) {
        epoch_task_spawner.spawn_task("worker network events", async move {
            loop {
                while let Some(event) = self.network_events.recv().await {
                    self.process_network_event(event);
                }
            }
        });
    }
```
When `self.network_events.recv().await` returns `None` (indicating channel closure), the inner `while let Some(event)` loop exits, but the outer loop immediately restarts. Since a closed channel continues returning None without blocking, this creates a tight spinning loop that consumes 100% CPU.

This prevents graceful node shutdown during upgrades or maintenance and causes resource exhaustion that can crash nodes

The `WorkerNetwork` is spawned during every epoch initialization,
```rust
WorkerNetwork::new(
            rx_event_stream,
            network_handle.clone(),
            consensus_config.clone(),
            *worker_id,
            validator,
        )
        .spawn(&epoch_task_spawner);
```
making this a critical system component.

The underlying `ConsensusNetwork` demonstrates proper shutdown handling using tokio::select! with shutdown signals
```rust
node_task_spawner.spawn_critical_task("Worker Network", async move {
           tokio::select!(
               _ = &node_shutdown => {
                   Ok(())
               }
               res = worker_network.run() => {
                   warn!(target: "epoch-manager", ?res, "worker network stopped");
                   res
               }
           )
       });
```
but the `WorkerNetwork` event loop lacks this protection.

**Recommendation**

Use `tokio::select!` to listen for both network events and shutdown signals



### [Informational-01] Unbounded RecordsDeque growth enables complete account lockout in `RecoverableWrapper` transfer system

**Description**

Every transfer operation enqueues a new record without any bounds checking.  
```solidity
uint256 rawIndex = _unsettledRecords[to].enqueue(amount, block.timestamp + recoverableWindow);
```
The enqueue function in the `RecordsDeque` implementation has no size limits and simply increments the tail counter indefinitely. 
```solidity
rd.tail++;
        rd.queue[rd.tail] = r;

        rawIndex = rd.tail;
    }
```
While self-transfers are prevented, an attacker can still execute this attack by transferring between different addresses they control. There are no minimum transfer amount restrictions in the codebase.

Several critical functions perform unbounded iterations over the deque:

`unsettledRecords()` function contains a while loop that iterates from tail to head of the deque 
```solidity
uint256 currentIndex = rd.tail;
        while (currentIndex != 0) {
            Record storage currentRecord = rd.queue[currentIndex];

            if (currentRecord.settlementTime > block.timestamp) {
                temp[count] = currentRecord;
                count++;
            }

            currentIndex = currentRecord.prev;
        }
```
`_unsettledBalanceOf()` function has multiple while loops that traverse records 
```solidity
while (current > cacheIndex) {
            Record memory r = _unsettledRecords[account].getAt(current);
            if (r.settlementTime <= block.timestamp) break; //it's expired
            unsettledTotal += r.amount;
            unsettledFrozen += r.frozen;
            current = r.prev;
        }

...

for (current = _unsettledRecords[account].head; current <= cacheIndex && current > 0;) {
            Record memory r = _unsettledRecords[account].getAt(current);
            if (r.settlementTime > block.timestamp) break; // not expired
            unsettledTotal -= r.amount;
            if (r.frozen > 0) {
                unsettledFrozen -= r.frozen;
            }
            current = r.next;
        }
```
The cleanup mechanism is throttled by MAX_TO_CLEAN  which limits how many records can be processed per transaction. This creates an asymmetry where records can be added faster than they can be cleaned, leading to unbounded growth.

This vulnerability can result in complete denial of service for recipient accounts and inability to transfer, unwrap, or query balances effectively locking of funds due to gas limitations

**Recommendation**

Implement a maximum deque size limit with forced cleanup to prevent unbounded growth while maintaining the recoverable wrapper's security guarantees.


### [Informational-02] Silent metrics registry fallback masks monitoring failures

**Description**

The Telcoin Network contains an observability bug in its Prometheus metrics initialization that silently masks monitoring failures. The issue occurs in multiple metrics structs including `Metrics`, `PrimaryChannelMetrics`, `PrimaryMetrics`, `WorkerMetrics`, and `MdbxMetrics` where the Default implementation uses an overly permissive error handling pattern.
```rust
match Self::try_new(default_registry()) {
            Ok(metrics) => metrics,
            Err(e) => {
                tracing::warn!(target: "tn::metrics", ?e, "Executor::try_new metrics error");
                // If we are in a test then don't panic on prometheus errors (usually an already
                // registered error) but try again with a new Registry. This is not
                // great for prod code, however should not happen, but will happen in tests due to
                // how Rust runs them so lets just gloss over it. cfg(test) does not
                // always work as expected.
                Self::try_new(&Registry::new()).expect("Prometheus error, are you using it wrong?")
            }
        }
```
When metrics registration fails with the `default_registry()`, the code logs a warning and falls back to creating a new `Registry::new()`. However, this fallback registry is never exposed via HTTP endpoints, which only serve metrics from the `default_registry()`.

This creates a dangerous scenario where:

- The metrics initialization appears successful, returning valid metric objects
- All metrics are registered in an unexposed registry, making them invisible to monitoring dashboards
- Dashboards show zeros/missing metrics while the application believes metrics are working
- Critical performance issues may go unnoticed due to missing observability
The same pattern exists across multiple metrics modules:

- `metrics.rs:262-274`
- `metrics.rs:530-542`
- `metrics.rs:148-160`
- `metrics.rs:58-68`

The metrics are used throughout the consensus system, including in the `ConsensusBus` where `ExecutorMetrics` are initialized

```rust
pub fn executor_metrics(&self) -> &ExecutorMetrics {
        &self.inner.executor_metrics
    }
```

**Recommendation**

Modify the Default implementations to panic or return an error when registry registration fails in production environments, rather than silently falling back to a new registry.


### [Informational-03] Stale early finalization flag due to node mode transition during batch fetching

A critical consensus bug exists in the Telcoin Network where the `early_finalize` flag becomes stale when a node's CVV (Committee Voting Validator) mode changes during the fetch_batches operation.
```rust
let early_finalize = if self.consensus_bus.node_mode().borrow().is_active_cvv() {
            // We are a CVV so we can finalize early.
            true
        } else {
            // Not a CVV so be more conservative about finalizing blocks.
            false
        };
```
The issue occurs in where the early_finalize flag is set once at the beginning of the batch fetching process based on the current node mode. However, the node mode can transition between `CvvActive` and `CvvInactive` states during the potentially time-consuming network operation of fetching batches from peers .

Node mode transitions occur in several scenarios:

- When a node detects it's out of sync during consensus
- When a node catches up and rejoins consensus

This creates a race condition where some blocks in a sub-DAG are processed with the wrong finalization behavior, leading to:

- Inconsistent block finality guarantees within the same sub-DAG
- Split-brain execution between peers processing the same blocks at different times
- Consensus safety violations where blocks are finalized when they shouldn't be (or vice versa)

The `early_finalize` flag controls a critical safety property - CVV nodes can safely finalize blocks immediately after execution, while non-CVV nodes must be more conservative to avoid advertising forked blocks.

**Recommendation**

Implement dynamic evaluation of the `early_finalize` flag for each batch or block within the sub-DAG instead of setting it once at the beginning of `fetch_batches`