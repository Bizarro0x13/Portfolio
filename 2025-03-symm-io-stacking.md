### [H-1] Missing constructor in the contracts.

**Description:** The `Vesting`, `SymmVesting` and `SymmStaking` contracts are missing constructor with `_disableInitializers` function, as written In the openzeppelin docs 

```comment
An uninitialized contract can be taken over by an attacker. This applies to both a proxy and its implementation contract, which may impact the proxy. To prevent the implementation contract from being used, you should invoke the _disableInitializers function in the constructor to automatically lock it when it is deployed:
```

**Impact:** The contract lacks proper initialization protection, allowing an attacker to reinitialize it and assign themselves the `ADMIN_ROLE`. This would enable them to transfer all contract tokens to their own account and perform other unauthorized actions.

**Proof of Concept:**
https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable

**Recommended Mitigation:** Add contructor with `_disableInitializers` function 

```solidity
constructor() {
    _disableInitializers();
}
```

### [L-1] Unsafe casting at `SymmVesting::_addLiquidity()`

**Description:** The `_addLiquidity()` function invokes `PERMIT2.approve`, where `symmIn` and `usdcIn` are unsafely downcasted to `uint160`, potentially leading to data loss or unintended behavior.

**Impact:** The router adds wrong amount of liquidity to the `ROUTER`.

**Proof of Concept:** https://github.com/sherlock-audit/2025-03-symm-io-stacking/blob/d7cf7fc96af1c25b53a7b500a98b411cd018c0d3/token/contracts/vesting/SymmVesting.sol#L187

**Recommended Mitigation:** Use openzeppelin's safeCast library to downcast `uint256` to `uint160`

### [L-2] `SymmVesting::_mintTokenIfPossible` does not account for tokens that are not `SYMM`

**Description:** The `_mintTokenIfPossible` function is called by the `_ensureSufficientBalance` function to mint tokens when the currentBalance of token is less than the withdraw amount, but the function does not take in account for other tokens as it will return without minting tokens.

**Impact:** This will cause functions like `Vesting::_claimUnlockedToken` and `Vesting::_claimLockedToken` to transfer wrong amount of tokens to the be transfered to the user, at last resulting in a revert.

**Proof of Concept:** https://github.com/sherlock-audit/2025-03-symm-io-stacking/blob/d7cf7fc96af1c25b53a7b500a98b411cd018c0d3/token/contracts/vesting/SymmVesting.sol#L258

**Recommended Mitigation:** Add a mechanism to revert when the token is not `SYMM` in the `_mintTokenIfPossible` and also move the `_ensureSufficientBalance` function call at the start of the `claimUnlockedToken` and `claimLockedToken` functions.