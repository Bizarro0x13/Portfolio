### [L-1] Missing check for launchTime in `MergeTgt::claimTitn`

**Description:** In `MergeTgt::claimTitn` function gets called from the user to claim Titn token before the 1 year waiting period is over. but the launchTime in the function is never getting checked. 


**Impact:** If the launchTime is not greater than zero then users cannot claim their tokens as the function will always revert.

**Proof of Concept:**

```js
function claimTitn(uint256 amount) external nonReentrant {

        require(amount <= claimableTitnPerUser[msg.sender], "Not enough claimable titn");

>>>     if (block.timestamp - launchTime >= 360 days) {
            revert TooLateToClaimRemainingTitn();
        }

        ...
}
```

**Recommended Mitigation:** 

```diff
function claimTitn(uint256 amount) external nonReentrant {

        require(amount <= claimableTitnPerUser[msg.sender], "Not enough claimable titn");

+       require(launchTime > 0, "Launch time not set");

         if (block.timestamp - launchTime >= 360 days) {
            revert TooLateToClaimRemainingTitn();
        }

        ...
}
```