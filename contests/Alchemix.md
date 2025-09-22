## Alchemix
[Contest details](https://cantina.xyz/code/e68909e6-3491-4a94-a707-ecf0c89cf72a/overview)

### [High-01] Zero-cost debt forgiveness in `_forceRepay()`

**Description**

In `AlchemistV3._forceRepay`, after reducing the user’s debt and collateral balance the contract executes:

```solidity
TokenUtils.safeTransfer(yieldToken, address(this), creditToYield);
```

Because this invokes an ERC-20 transfer from the Alchemist contract to itself, no tokens actually leave the vault. The user’s on-chain debt is forgiven and their recorded collateral balance is decremented, but the corresponding yield tokens remain locked in the contract. This “zero-cost” forgiveness:

- Poisons the protocol’s accounting (totalDebt and cumulativeEarmarked decrease, but collateral stays)
- Breaks later redemptions (Transmuter cannot pull the expected yield tokens)
- Enables a helper to repeatedly liquidate an underwater account at no token cost, draining its debt while leaving collateral behind

Based on the natspec the yield tokens are meant to be sent to the `Transmuter`

**Recommendation**

Change the self-transfer to send real tokens out of the vault. For example:

```diff
- TokenUtils.safeTransfer(yieldToken, address(this), creditToYield);
+ TokenUtils.safeTransfer(yieldToken, transmuter, creditToYield);
```

### [High-02] Global DoS when all debt becomes earmarked

**Description**

In the `_earmark()` function, the contract computes an “earmark weight” increment using `PositionDecay.WeightIncrement(amount, totalDebt - cumulativeEarmarked)`.

Once `cumulativeEarmarked` equals `totalDebt`, the second argument becomes zero while amount (queried from the Transmuter) can still be ≥ 1.

Inside `WeightIncrement`, a `require(increment <= total)` check fails (since `increment > 0` but `total == 0`), causing every subsequent call to any state-mutating function (all of which invoke `_earmark()`) to revert unconditionally. This permanently locks the contract once the final wei of debt is earmarked.

**Recommendation**

Before calling `WeightIncrement`, check if there is any unearmarked debt:

```solidity
if (totalDebt == 0 || cumulativeEarmarked == totalDebt) {
    return;
}
```

so you never attempt to weight-increment when `totalDebt - cumulativeEarmarked == 0`


### [High-03] Fee unit mismatch lets liquidators siphon collateral

**Description**

In `AlchemistV3._liquidate`, the third return value baseFee from `calculateLiquidation` is denominated in underlying tokens, but the code treats it as debt-token units before converting to yield tokens:

```solidity
// @audit - baseFee is in underlying tokens…
(uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee) = calculateLiquidation(...);

// @audit…but here it is treated as debt tokens:
feeInYield = convertDebtTokensToYield(baseFee);
```

Because `convertDebtTokensToYield` first normalizes from debt to underlying and then to yield, this double-misinterpretation causes liquidators to receive an incorrect (often vastly inflated) yield token payout when underlying ≠ debt (e.g. different decimals or de-pegged synthetics) .

Furthermore, the subsequent `feeBonus` withdrawal from the fee vault is done in underlying units—again assuming 1:1 parity with debt—amplifying the mismatch and enabling a liquidator to drain all the protocol’s yield tokens with just a few liquidations

**Recommendation**

Convert the base fee (underlying) directly to yield

```diff
- feeInYield = convertDebtTokensToYield(baseFee);
+ feeInYield = convertUnderlyingTokensToYield(baseFee);
```

### [High-04] Double spend and treasury drain in repay() function

**Description**

In the `repay()` function, the contract pulls the full creditToYield amount from the user and then immediately drains the same full amount from the protocol’s idle balance into the fee receiver. The relevant code is

```solidity
TokenUtils.safeTransferFrom(yieldToken, msg.sender, transmuter, creditToYield);
TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, creditToYield);
```

Because the entire repayment is treated as a fee, an attacker can:

1. Deposit minimal collateral and mint a small debt.
2. Call repay(1) (or any tiny amount) in a loop.
3. Each iteration, pull creditToYield from themselves and siphon the same amount out of the protocol into the fee vault.
4. Continue until the contract’s yield-token balance is zero.
5. At that point, all future redemptions and liquidations revert effectively DoSing the system.

**Recommendation**

Ensure only the intended fee percentage is transferred to the fee receiver and the net repayment goes to the transmuter

### [High-05] Protocol fee is silently orphaned in `burn()`

**Description**

In `burn()`, the user’s collateral balance is reduced by the protocol fee, but no tokens are ever moved to the fee receiver. The offending line:

```solidity
// Deducts fee from user’s collateral accounting…
_accounts[recipientId].collateralBalance -= convertDebtTokensToYield(credit) * protocolFee / BPS;
```

Because there is no accompanying safeTransfer to protocolFeeReceiver, the fee amount remains stranded in the contract’s yield-token balance:

- Fee receiver never accrues revenue.
- TVL overstated. Orphaned tokens inflate the apparent liquidity.
- Future operations may revert. Withdrawals or redemptions that depend on available free collateral can fail when they attempt to pull from a user’s balance that was already debited but not actually moved.

**Recommendation**

After deducting the fee from the user’s collateral balance, transfer it to the designated receiver



### [Medium-01] Division-by-zero in bad-debt scaling freezes Transmuter redemptions

**Description**

In `Transmuter.claimRedemption`, the code computes:

```solidity
// Ratio of total synthetics issued by the alchemist / underlying value of collateral
uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.yieldToken()) / alchemist.getTotalUnderlyingValue();
```

Here, if the Alchemist vault’s total underlying value ever becomes zero (for instance, after everyone withdraws collateral but before any new deposits arrive), the division by zero will revert.

Because every call to claimRedemption recomputes badDebtRatio, all redemptions become permanently impossible until at least 1 wei of collateral is returned to the vault. This is a system-wide DoS on the redemption mechanism

**Recommendation**

Guard against a zero denominator before dividing

```diff
function claimRedemption(uint256 id) external {
     // … earlier logic …

-    uint256 badDebtRatio =
-        alchemist.totalSyntheticsIssued()
-        * 10**TokenUtils.expectDecimals(alchemist.yieldToken())
-        / alchemist.getTotalUnderlyingValue();
+    uint256 totalUnderlying = alchemist.getTotalUnderlyingValue();
+    require(totalUnderlying > 0, "Transmuter: no collateral available");
+
+    uint256 badDebtRatio =
+        alchemist.totalSyntheticsIssued()
+        * 10**TokenUtils.expectDecimals(alchemist.yieldToken())
+        / totalUnderlying;

     if (badDebtRatio > FIXED_POINT_SCALAR) {
         claimAmount = claimAmount * FIXED_POINT_SCALAR / badDebtRatio;
         feeAmount   = feeAmount   * FIXED_POINT_SCALAR / badDebtRatio;
     }

     // … rest of function …
 }
 ```

 ### [Medium-02] Liquidator can claim duplicate fees

 **Description**

 Liquidators receive the liquidatorFee twice on each liquidation: Base fee (baseFee) is computed in `calculateLiquidation()` as

```solidity
fee = (surplus * feeBps) / BPS;
```

and paid out in yield tokens via

```solidity
TokenUtils.safeTransfer(yieldToken, msg.sender, feeInYield);
```
Bonus fee (feeBonus) is then re-computed as

```solidity
feeBonus = debtToBurn * liquidatorFee / BPS;
```
and paid out in underlying tokens from the alchemistFeeVault via
```solidity
IFeeVault(alchemistFeeVault).withdraw(msg.sender, feeInUnderlying);
```
Because both calculations use the same liquidatorFee, an attacker can open tiny under-collateralized positions (or liquidate their own), trigger the liquidation, and repeatedly harvest both fee payouts eventually draining the fee vault.

**Recommendation**

Unify the fee logic so that liquidatorFee is applied once per liquidation, with any split between yield and underlying made explicit.

