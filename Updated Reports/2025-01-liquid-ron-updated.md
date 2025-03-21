### [H-1] Calculation of totalAssets is wrong in `LiquidRon::totalAssets()` when operator fee is claimed.

**Description:** In `LiquidRon::totalAssets()` the totalAssets in the contract is calculated by adding totalAssets totalStakedAmount and totalRewards
```js
    function totalAssets() public view override returns (uint256) {
        return super.totalAssets() + getTotalStaked() + getTotalRewards();
    }
```
The operator fee is calculates when the user harvests the rewards and the operatorFee is added the `operatorFeeAmount` storage variable, the harvested amount from the user stores the operatorFee but does not exclude the fee when calculating totalAssets leading to wrong calculation of user's shares.

**Impact:** Potential assets loss for the users who withdraw funds after operator withdraw the fee.

**Proof of Concept:**
1. The operator call the `harvest()`. This will increase the WRON balance in owned by the vault and also increase `operatorFeeAmount`.
2. The new user deposit assets and receive shares. The calculation of totalAssets() will include the amount of operator's fee.
3. The operator withdraw the fee by calling fetchOperatorFee() function
4. The new user withdraw his funds by calling redeem(). Now the user receive less assets because the calculation of totalAssets will be based on the new WRON balance after fee withdrawal.

**Proof of Concept(Coded):** Copy the POC code below to `LiquidRonTest` contract in `test/LiquidRon.t.sol` and then run the test.

```js
function test_withdraw_new_user() public {
        address userA = vm.addr(24);
        address userB = vm.addr(25);

        uint256 amount = 100000 ether;
        vm.deal(userA, amount);
        vm.deal(userB, amount);

        vm.prank(userA);
        liquidRon.deposit{value: amount}();

        uint256 delegateAmount = amount/7;
        uint256[] memory amounts = new uint256[](5);
        for(uint256 i=0; i<5; i++) {
            amounts[i] = delegateAmount;
        }
        liquidRon.delegateAmount(0, amounts, consensusAddrs);

        skip(86400 * 365 + 2 + 1);
        // operator fee before harvest
        assertTrue(liquidRon.operatorFeeAmount() == 0);
        liquidRon.harvest(0, consensusAddrs);
        // operatorFee after harvest
        assertTrue(liquidRon.operatorFeeAmount() > 0);

        // new user deposit
        vm.prank(userB);
        liquidRon.deposit{value: amount}();
        uint256 userBShares = liquidRon.balanceOf(userB);
        uint256 expectedRedeemAmount = liquidRon.previewRedeem(userBShares);

        // fee withdrawal by operator
        liquidRon.fetchOperatorFee();
        assertTrue(liquidRon.operatorFeeAmount() == 0);

        vm.prank(userB);
        liquidRon.redeem(userBShares, userB, userB);

        console.log(userB.balance);
        console.log(expectedRedeemAmount);
        assertTrue(userB.balance == expectedRedeemAmount);

    }
```

Based on the POC code above, the last assertion `assertTrue(user2.balance == expectedRedeemAmount);` will fail because the amount withdrawn is not equal to the expected withdrawn.

**Recommended Mitigation:** Change the formula that calculate `totalAssets()` to include `operatorFeeAmount` to subtract the total balance.

```diff
function totalAssets() public view override returns (uint256) {

-        return super.totalAssets() + getTotalStaked() + getTotalRewards();

+        return super.totalAssets() + getTotalStaked() + getTotalRewards() - operatorFeeAmount;

    }
```

### [M-1] User can earn rewards by frontrunning the new rewards accumulation in Ron staking without actually delegating his tokens

**Description:** The Ron staking contract let us earn rewards by delegating our tokens to a validator. But you will only earn rewards on the lowest balance of the day (source). So if you delegate your tokens on the first day, you are going to earn 0 rewards for that day as your lowest balance was 0 on that day. This will happens with every new delegator.

Now the issue with LiquidRon is that, there will be many users who will be depositing their tokens in it. And there is no such kind of time or amount restriction for new delegators if some people have already delegated before them. So with direct delegation, the new rewards flow will be this:

User -> delegate -> RonStaking -> Wait atleast a day -> New Rewards
But if we deposit through LiquidRon it has become this:

User -> LiquidRon -> LiquidProxy -> New Rewards
Now a user can earn rewards by just depositing the tokens into the LiquidRon by frontrunning the new rewards arrival and immediately withdraw them. But if a user try to do this by frontrunning the LiquidRon::harvest(...) call, this will not be possible. Because when he deposits, he will get shares in return which will already be accounting for any unclaimed rewards through getTotalRewards(...) in LiqiuidRon::totalAssets(...):

```js
    function totalAssets() public view override returns (uint256) {
@>        return super.totalAssets() + getTotalStaked() + getTotalReward();
    }
```
But instead of frontrunning this harvest(...) call, a user can just frontrun the new rewards arrival in the RonStaking contract itself. Because as per the Ronin staking docs, a user will be able to claim new rewards after every 24 hours.

Also, if we look at the _getReward(...) (used by claimRewards etc.) function of the Ronin staking contract, the rewards will be same as before as long as we are not in new period:

```js
  function _getReward(
    address poolId,
    address user,
    uint256 latestPeriod,
    uint256 latestStakingAmount
  ) internal view returns (uint256) {
    UserRewardFields storage _reward = _userReward[poolId][user];
@>    if (_reward.lastPeriod == latestPeriod) {
@>      return _reward.debited;
    }
```
So if a user frontrun before this, the getTotalRewards(...) function will also not account for new rewards as long as we are not in new period.

Also note that, if a user frontruns and deposit, he didn’t not actually delegated his tokens as the tokens are only delegated once the operator calls LiquidRon::delegateAmount(...) function:

```js
    function delegateAmount(uint256 _proxyIndex, uint256[] calldata _amounts, address[] calldata _consensusAddrs)
        external
        onlyOperator
        whenNotPaused
    {
        address stakingProxy = stakingProxies[_proxyIndex];
        uint256 total;
        if (stakingProxy == address(0)) revert ErrBadProxy();
        // @audit how many max validators can be there?
        for (uint256 i = 0; i < _amounts.length; i++) {
            if (_amounts[i] == 0) revert ErrNotZero();
            _tryPushValidator(_consensusAddrs[i]);
            total += _amounts[i];
        }
@>        _withdrawRONTo(stakingProxy, total);
@>        ILiquidProxy(stakingProxy).delegateAmount(_amounts, _consensusAddrs);
    }
```
So a user is just depositing, claiming and withdrawing his tokens but not actually delegating.

**Proof of Concept:**
There are currently millions of delegated tokens deposited into the LiquidRon which are delegated to many validators through LiquidProxy. And each day the LiquidRon is earning let’s 1000 tokens as a reward.
But these rewards are only claimable in the new epoch.
So a user keeps an eye on the new rewards arrival into the RonStaking for the LiquidRon and also on the new epoch update.
Once rewards are arrived he waits for the epoch update, and frontruns this epoch update and deposits into the LiquidRon.
Once this new epoch is updated, these new rewards will be starting to reflect into the the share price through LiquidRon::totalAssets(...).
User withdraws immediately with the profit.
In the above case, the user didn’t have to stake his tokens and still earns the share of the rewards. Also, this can be done each and every time to claim the rewards.

He also didn’t have to bear any risk of loss in case something bad happens (maybe some kind of attack) and the share price value goes down; while others will also have to bear this loss.

**Recommended Mitigation:** Incorporate some kind of locking mechanism for new depositor like Ron staking contract does. Or, go for some alternate option.

### [M-2] Operators are unable to perform any actions due to incorrect modifier implementation

**Description:** The onlyOperator modifier in LiquidRon contract is intended to restrict access to specific functions to either the owner or operator. Functions like harvest, delegateAmount, and harvestAndDelegateRewards rely on this modifier for access control.

However, the modifier implementation is incorrect and will always revert when called by any address other than the owner, even if the caller is a valid operator. As a result, operators are completely unable to perform any of their intended actions.

As we can see, if msg.sender is not the owner, the first condition evaluates to true, triggering a revert. Even if we ignore the first condition, when an operator calls a function using this modifier, operator[msg.sender] evaluates to true, causing the revert to be triggered again.

```js
   /// @dev Modifier to restrict access of a function to an operator or owner
    modifier onlyOperator() {
        if (msg.sender != owner() || operator[msg.sender]) revert ErrInvalidOperator();
        _;
    }
```

**Proof of Concept:**
Paste the following test in LquidRon.operator.t.sol:

```js
    function test_wronModifierIImplementation() public {
        address operator = makeAddr("operator");
        liquidRon.updateOperator(operator, true);
        assertTrue(liquidRon.operator(operator));
        vm.startPrank(operator);
        vm.expectRevert(LiquidRon.ErrInvalidOperator.selector);
        liquidRon.harvest(1, consensusAddrs);
    }
```
**Recommended Mitigation:** 
```diff
    /// @dev Modifier to restrict access of a function to an operator or owner
    modifier onlyOperator() {
-       if (msg.sender != owner() || operator[msg.sender]) revert ErrInvalidOperator();
+       if (msg.sender != owner() && !operator[msg.sender]) revert ErrInvalidOperator();
        _;
    }
```