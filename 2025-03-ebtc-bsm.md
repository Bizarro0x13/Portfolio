### [M-1] Wrong amount parameter when withdrawing `_assetAmount` in `ERC4626Escrow::_onWithdraw()`

**Description:** The `ERC4626Escrow::_onWithdraw()` function takes `_assetAmount` as input but relies on `_ensureLiquidity` to calculate the actual amount that can be redeemed `(redeemedAmount)`. However, it still calls B`aseEscrow::_onWithdraw()` to withdraw `_assetAmount`, which may be more than what the contract can actually redeem.

**Impact:** This will cause the contract to assume full withdrawal was successful, even when actual liquidity is insufficient.
When The User calls the `_buyAsset` function, then the function will revert with `InsufficientAssetTokens` error.

**Proof of Concept:**
1. An Attacker calls the `buyAsset` function with `_ebtcAmountIn == totalAssetsDeposited`
2. The function calls the `ERC4626Escrow::_onWithdraw()` which ensures their is enough liquidity
3. If the Liquidity is not sufficient, the contract sells shares based on the protocol's `EXTERNAL_VAULT` shares.
4. The `_amountRedeemed` comes out to be less than the `_amountRequired`. But the function calls the `BaseEscrow::_withdraw` and reduce the `totalAssetsDeposited` to 0
5. If another user wants to `buyAsset`, the function will revert since no assets remain available for withdrawal.

**Recommended Mitigation:** 

```diff
 function _onWithdraw(
        uint256 _assetAmount
    ) internal override returns (uint256) {
        uint256 redeemedAmount = _ensureLiquidity(_assetAmount);
-       super._onWithdraw(_assetAmount);
+       super._onWithdraw(redeemedAmount);
        return redeemedAmount;
    }
```