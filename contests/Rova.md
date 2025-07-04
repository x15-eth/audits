## Rova
[Contest Details](https://audits.sherlock.xyz/contests/498/report)

### [Medium-01] Incorrect Token Amount Adjustment in `updateParticipation`

**Summary**

The `updateParticipation` function adjusts the user's total token commitment (`userTokensByLaunchGroup`) using currency amounts instead of token deltas. This leads to incorrect calculations, potential underflows, and invalid token tracking

**Vulnerability Details**
When a user first participates, the contract updates their total token commitment in the `_userTokensByLaunchGroup` mapping using the token amount they requested. For example, in the participate function, it does:

```solidity
// Calculate new total tokens requested for the user  
uint256 newUserTokenAmount = userTokenAmount + request.tokenAmount;  
....  
  
// Update the mapping with the token amount  
userTokens.set(request.userId, newUserTokenAmount);  
```
  
This mapping is meant to track token amounts that the user has committed.

In `updateParticipation`, the new currency amount is calculated based on the updated token request:

```solidity
uint256 newCurrencyAmount = _calculateCurrencyAmount(tokenPriceBps, request.tokenAmount);  
```

Here, `_calculateCurrencyAmount` converts the token amount into a currency amount using the token price.

The function then compares the previous currency amount (`prevInfo.currencyAmount`) with the new currency amount (`newCurrencyAmount`). Depending on whether the new amount is lower or higher, it computes a delta in currency terms:

But the problem is that the `_userTokensByLaunchGroup` mapping is meant to record token amounts (like 100 tokens), but here it is being adjusted by values in currency units (e.g., dollars or another ERC20 denomination).

You cannot subtract or add a currency delta (calculated as `prevInfo.currencyAmount - newCurrencyAmount`) from a token amount, because these two quantities are not directly comparable.

**Impact**

This can lead to underflows, incorrect total token tracking, and ultimately an invalid record of the user’s token commitment

**Recommendations**

The function should adjust the user's token commitment based on the difference in token amounts, not the difference in currency amounts

**Code Snippets**

https://github.com/sherlock-audit/2025-02-rova/blob/main/rova-contracts/src/Launch.sol#L361
https://github.com/sherlock-audit/2025-02-rova/blob/main/rova-contracts/src/Launch.sol#L374