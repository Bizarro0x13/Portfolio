### [M-1] Wrong calculation of `borrowingRate` in `InterestRateUtils::calculateBorrowingRate`

**Description:** The `InterestRateUtils::calculateBorrowingRate` function calculates the borrowingRate based on the `utilizationRate` and `config`. In the else block where the `utilization > config.utilizationB` the borrowingRate calculation is incorrect causing the transaction to revert.

```solidity
function calculateBorrowingRate(DataTypes.InterestRateConfig storage config, uint256 utilizationRate)
        internal
        view
        returns (uint256 borrowingRate)
    {
        if (utilizationRate <= config.utilizationA) {
            ...
        } else if (utilizationRate <= config.utilizationB) {
            ...
        } else {
            ...

            borrowingRate = (uint256(config.maxBorrowingRate) - config.borrowingRateB)
>>>             * (utilizationRate - config.utilizationB) / (PRECISION - config.utilizationB) + config.borrowingRateB;
        }
        return borrowingRate;
    }
```

**Impact:** The function will always revert when the `utilization > config.utilizationB` 

**Proof of Concept:** https://github.com/sherlock-audit/2025-02-yieldoor-bizarro0x13/blob/49288a6ac9f20bd57768ca475bea22d577db23e2/yieldoor/src/libraries/InterestRateUtils.sol#L41

**Recommended Mitigation:** 
```diff
function calculateBorrowingRate(DataTypes.InterestRateConfig storage config, uint256 utilizationRate)
        internal
        view
        returns (uint256 borrowingRate)
    {
        if (utilizationRate <= config.utilizationA) {
            ...
        } else if (utilizationRate <= config.utilizationB) {
            ...
        } else {
            ...

            borrowingRate = (uint256(config.maxBorrowingRate) - config.borrowingRateB)
-             * (utilizationRate - config.utilizationB) / (PRECISION - config.utilizationB) + config.borrowingRateB;
+             * (config.utilizationB - utilizationRate) / (PRECISION - config.utilizationB) + config.borrowingRateB;
        }
        return borrowingRate;
    }
```


### [L-1] In `Vault` contract `withdraw` and `_calDeopsit` is overshadowing the totalSupply function present in the ERC20 contract.

**Recommended Mitigation:** choose a different name for the local variable.