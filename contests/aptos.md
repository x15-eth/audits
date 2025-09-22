## Aave-aptos
[Contest details](https://cantina.xyz/code/ad445d42-9d39-4bcf-becb-0c6c8689b767/overview)

### [Medium-01] Missing chainlink feed freshness check

**Description**

The `get_asset_price_internal` function retrieves the current price for an asset via Chainlink but does not verify how recently that feed was updated.

```move
fun get_asset_price_internal(asset: address): u256 acquires PriceOracleData {
    // … after confirming a feed exists …
    let benchmarks =
        chainlink_router::get_benchmarks(
            &get_resource_account_signer(),
            vector[feed_id],
            vector[]
        );
    assert_benchmarks_match_assets(vector::length(&benchmarks), 1);
    let benchmark = vector::borrow(&benchmarks, 0);
    let price = chainlink::get_benchmark_value(benchmark);
    validate_oracle_price(price);
    return price;
}
```
Because there is no timestamp verification, a Chainlink feed that has not been updated for an extended period could return an outdated price. The protocol would then use that stale value for lending, borrowing, collateral valuations, or liquidations.

**Recommendation**

Introduce a staleness threshold and enforce it immediately after fetching the Chainlink benchmark.


### [Low-01] Underlying tokens permanently locked in aToken resource account

**Description**

The `rescue_tokens` entrypoint in `aave_pool::a_token_factory` explicitly forbids rescuing the underlying asset:

```move
assert!(
    token != token_data.underlying_asset,
    error_config::get_eunderlying_cannot_be_rescued()
);
```
Because no other public function exists to withdraw the underlying from the aToken’s resource account, any underlying tokens mistakenly sent there (user error) become irrecoverably trapped.

Users assume the protocol admin can always sweep misplaced funds, but this check blocks that path entirely.

**Recommendation**

Modify `rescue_tokens` to allow rescuing the underlying asset, but require an explicit multi-signature or time-locked admin privilege check to prevent abuse.


### [Low-02] Redundant price computation in is_asset_price_capped

**Description**

The is_asset_price_capped function currently calls` get_asset_price_internal(asset)` and then immediately calls `get_asset_price(asset)`, which itself invokes `get_asset_price_internal(asset)` again. This leads to two consecutive on‐chain reads of the same price data:
```move
public fun is_asset_price_capped(asset: address): bool acquires PriceOracleData {
    let price_oracle_data = borrow_global<PriceOracleData>(oracle_address());
    if (!smart_table::contains(&price_oracle_data.capped_assets_data, asset)) {
        return false;
    };
    // First call to fetch base price
    get_asset_price_internal(asset)
        // Second call loads base price again inside get_asset_price
        > get_asset_price(asset)
}
```
Every call to `get_asset_price_internal` can involve a Chainlink lookup or a custom‐price read. Performing it twice in one function doubles that cost.

**Recommendation**

Compute the “base price” once, store it in a local variable, and compare it against the cap.
```move
public fun is_asset_price_capped(asset: address): bool acquires PriceOracleData {
    let price_oracle_data = borrow_global<PriceOracleData>(oracle_address());
    if (!smart_table::contains(&price_oracle_data.capped_assets_data, asset)) {
        return false;
    };
    // Fetch the base price exactly once
    let base_price = get_asset_price_internal(asset);
    let cap = *smart_table::borrow(&price_oracle_data.capped_assets_data, asset);
    base_price > cap
}
```

### [Low-03] Shadowed pending_ltv_set causes misleading event payload

**Description**

In `pool_configurator.set_reserve_freeze`, the variable pending_ltv_set is declared twice—once outside the if (freeze) block and then again inside it—causing the inner value to shadow the outer one.
```move
let pending_ltv_set = 0;                // outer variable (event payload)
…
if (freeze) {
    let pending_ltv_set = reserve_config::get_ltv(&reserve_config_map); // shadows outer
    smart_table::upsert(&mut internal_data.pending_ltv, asset, pending_ltv_set);
    reserve_config::set_ltv(&mut reserve_config_map, 0);
} else {
    …
};
event::emit(PendingLtvChanged { asset, pending_ltv_set });
```
Because the `event::emit` refers to the outer `pending_ltv_set`, which was initialized to 0, the event always reports `pending_ltv_set = 0` even when the inner block stored a nonzero LTV in storage.

Downstream indexers, dashboards, and risk‐monitoring tools that rely on the `PendingLtvChanged` event will therefore receive incorrect data whenever a reserve is frozen, misleading any off‐chain systems about the true pending LTV value.

**Recommendation**

Remove the inner let and instead update the outer `pending_ltv_set` so that the event emits the correct LTV value.


### [Low-04] Isolation-mode debt-ceiling rounding lets borrowers slip in < 1 % extra debt per ceiling unit

**Description**

When a reserve is in isolation mode, every new borrow is converted to “ceiling-units” with a floor division
```move
//validation_logic.move
let isolation_mode_total_debt =
    pool::get_reserve_isolation_mode_total_debt(mode_collateral_reserve_data) as u256;
let total_debt =
    isolation_mode_total_debt
      + (amount
         / math_utils::pow(
             10,
             (reserve_decimals
                - reserve_config::get_debt_ceiling_decimals())
         )
        );
assert!( total_debt <= isolation_mode_debt_ceiling, … );
```
Because the division truncates, a borrower can repeatedly request amounts that sit just below the next multiple of scale. Each call adds zero to isolation_mode_total_debt, yet increases real exposure by up to scale − 1 smallest units.

Thus, at most < 1 % of one ceiling-unit escapes counting per borrow. While individually small, scripted borrowers can:

- Over-allocate risk margin – e.g. a 1 000 000-unit ceiling allows ~9.999 990 extra tokens of an 18-dec asset, multiplied across many borrowers.
- Grief the market – fill the ceiling earlier than expected, preventing later users from borrowing the “last sat.”

**Recommendation**

Round up instead of down


### [Informational-01] No event emitted on rescue_tokens

**Description**

The `rescue_tokens` entrypoint in `aave_pool::a_token_factory` allows an admin to transfer out non-underlying tokens from an aToken’s resource account, but it does not emit any event when this transfer occurs.

As a result, off-chain indexers and monitoring tools have no reliable on-chain signal for when or how much was rescued

And there is no corresponding “TokensRescued” event defined in the protocol’s events module

**Recommendation**

Emit an event whenever tokens are rescued from an aToken account.


### [Informational-02] Double storage reads in `balance_of`

**Description**

The `balance_of` view in `aave_pool::a_token_factory` performs two separate on-chain borrows when a single lookup would suffice:

First it calls `scaled_balance_of`, which does a `borrow_global<TokenMap>`.

Then it calls get_underlying_asset_address, which does another `borrow_global<TokenMap>` (and `TokenData`). Each `borrow_global` has a nontrivial gas and CPU cost; doing both doubles that overhead for every single balance_of call

**Recommendation**

Refactor `balance_of` to perform a single borrow and then extract the needed fields.


### [Informational-03] Insufficient event emission on role creation

**Description**

In grant_role_internal, when a role does not yet exist, the function both creates the new role entry and grants the first member.
```move
if (!has_role(role, user)) {
    let role_res = get_roles_mut();
    if (!smart_table::contains(&role_res.acl_instance, role)) {
        let members = acl::empty();
        let role_data = RoleData { members, admin_role: default_admin_role() };
        smart_table::add(&mut role_res.acl_instance, role, role_data);
    } else {
        let role_data = smart_table::borrow_mut(&mut role_res.acl_instance, role);
        acl::add(&mut role_data.members, user);
    };

    // @ Emits only RoleGranted, even on first‐time role creation
    event::emit(RoleGranted { role, account: user, sender: signer::address_of(admin) });
}
```
However, it emits only a RoleGranted event. This conflates “role creation” and “role assignment” into a single event, making it impossible for off‐chain listeners or audit tools to distinguish between adding a member to an existing role versus creating that role for the first time.

As a result, dashboards or audit logs will incorrectly show that a brand‐new role existed prior to its first grant.

**Recommendation**

Emit a separate `RoleCreated` event whenever a new role is created, in addition to the `RoleGranted` event.


### [Informational-04] Zero-amount flash loans are allowed

**Description**

The `validate_flashloan_simple` function only checks that the total aToken supply is at least the requested amount:
```move
let a_token_total_supply =
    a_token_factory::total_supply(pool::get_reserve_a_token_address(reserve_data));
assert!(a_token_total_supply >= amount, error_config::get_einvalid_amount());
```
Because it never asserts `amount != 0`, a caller can request a zero-amount flash loan. The downstream logic in `execute_flash_loan_simple` will still run, updating state and emitting events without transferring any funds
```move
validation_logic::validate_flashloan_simple(reserve_data, amount);
// …
let virtual_balance = pool::get_reserve_virtual_underlying_balance(reserve_data);
virtual_balance = virtual_balance - (amount as u128);
pool::set_reserve_virtual_underlying_balance(reserve_data, virtual_balance);
// …
transfer_underlying_to(&flashloan);
```

**Recommendation**

Add an explicit check for `amount > 0` in `validate_flashloan_simple`.