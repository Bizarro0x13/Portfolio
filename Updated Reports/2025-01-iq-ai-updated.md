### [S-#] TITLE (Root Cause + Impact)

**Description:** 

**Impact:** 

**Proof of Concept:**

**Recommended Mitigation:** 

### [H-1] Adversary can win proposals with voting power as low as 4%

**Description:** The expected quorum for proposals is 25% of the voting power.

An attacker can execute any proposal with as low as 4% of the voting power.

Proposals that don’t get enough quorum are expected to fail because of that threshold, but the bug bypasses that protection by a 6.25x lower margin. Deeming High severity as a form of executing malicious proposals against expectations.

**Proof of Concept:** The error is in the 4 in the line GovernorVotesQuorumFraction(4). It doesn’t represent 1/4th of supply but 4/100 actually.

```js
    constructor(
        string memory _name,
        IVotes _token,
        Agent _agent
    )
        Governor(_name)
        GovernorVotes(_token)
@>      GovernorVotesQuorumFraction(4) // quorum is 25% (1/4th) of supply
    {
        agent = _agent;
    }
```

Ref: https://github.com/code-423n4/2025-01-iq-ai/blob/main/src/TokenGovernor.sol#L55

This can be seen in the GovernorVotesQuorumFraction OpenZeppelin contract that is inherited.

Note how the quorumDenominator() is 100 by default and how the quorum() is calculated as supply * numerator / denominator.

In other words, 4% for the protocol governor (instead of 25%).

```js
/**
 * @dev Returns the quorum denominator. Defaults to 100, but may be overridden.
 */
function quorumDenominator() public view virtual returns (uint256) {
    return 100;
}
/**
 * @dev Returns the quorum for a timepoint, in terms of number of votes: `supply * numerator / denominator`.
 */
function quorum(uint256 timepoint) public view virtual override returns (uint256) {
    return (token().getPastTotalSupply(timepoint) * quorumNumerator(timepoint)) / quorumDenominator();
}
```
Ref: https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v5.2.0/contracts/governance/extensions/GovernorVotesQuorumFraction.sol#L62-L74

**Recommended Mitigation:** 

```diff
constructor(
        string memory _name,
        IVotes _token,
       Agent _agent
    )
        Governor(_name)
        GovernorVotes(_token)
-       GovernorVotesQuorumFraction(4) // quorum is 25% (1/4th) of supply
+       GovernorVotesQuorumFraction(25) // quorum is 25% (1/4th) of supply
    {
        agent = _agent;
    }
```

### [M-1] Anyone can deploy a new FraxSwapPair with a Low fee incurring losses to the protocol

**Description:** The `LiquidityManager::moveLiquidity` function is responsible for moving liquidity to a FraxSwapPair, If a FraxSwapPair does not already exist, it creates a new one with a default fee of 1%. If a pair already exists, it transfers liquidity to that pair.
A malicious user can deploy a `FraxSwapPair` with a very low fee (e.g., 0.01%) before the `moveLiquidity` function is called. When the `moveLiquidity` function is executed, it will use this low-fee pair instead of creating a new one with the default 1% fee. This results in a loss of fees for the protocol, as swaps will now incur a much lower fee.

**Proof of Concept:** Change the Fee in FraxSwapPair deployment from 1 to 100 to change fee from 0.01% to 1%:

```js
function test_low_fee() public {
        setUpFraxtal(12_918_968);
        address whale = 0x00160baF84b3D2014837cc12e838ea399f8b8478;
        uint256 targetCCYLiquidity = 6_100_000e18;
        factory = new AgentFactory(currencyToken, 0);
        factory.setAgentBytecode(type(Agent).creationCode);
        factory.setGovenerBytecode(type(TokenGovernor).creationCode);
        factory.setLiquidityManagerBytecode(type(LiquidityManager).creationCode);
        //factory.setTargetCCYLiquidity(1000e18);
        factory.setInitialPrice(0.1e18);
        vm.startPrank(whale);
        currencyToken.approve(address(factory), 1e18);
        agent = factory.createAgent("AIAgent", "AIA", "https://example.com", 0);
        token = agent.token();
        // Buy from the bootstrap pool
        manager = LiquidityManager(factory.agentManager(address(agent)));
        bootstrapPool = manager.bootstrapPool();
        currencyToken.approve(address(bootstrapPool), 10_000_000e18);
        bootstrapPool.buy(6_000_000e18);
vm.stopPrank();
        // the above code is setup()
// griefer buys from `BootstrapPool` to get minimum liquidity required to create a pool
        address _griefer = makeAddr("griefer");
        deal(address(currencyToken), _griefer, 1000 ether);
        vm.deal(_griefer, 10 ether);
        vm.startPrank(_griefer);
        currencyToken.approve(address(bootstrapPool), 10 ether);
        uint256 tknamt = bootstrapPool.buy(5 ether);
//griefer creates `FraxSwapPair`
        IFraxswapPair fraxswapPair =
            IFraxswapPair(manager.fraxswapFactory().createPair(address(currencyToken), address(token), 1)); // Fee set here
        currencyToken.transfer(address(fraxswapPair), 5 ether);
        token.transfer(address(fraxswapPair), tknamt);
        fraxswapPair.mint(address(_griefer));
// Move liquidity
        manager.moveLiquidity();
        // Swap from the Fraxswap pool
        //token(0) is AI token
       //token(1) is currency token
       // griefer swaps 1 ether worth of his currency token for AI token
        fraxswapPair = IFraxswapPair(manager.fraxswapFactory().getPair(address(currencyToken), address(token)));
        uint256 amountOut = fraxswapPair.getAmountOut(1e18, address(currencyToken));
        currencyToken.transfer(address(fraxswapPair), 1e18);
        if (fraxswapPair.token0() == address(currencyToken)) {
            fraxswapPair.swap(0, amountOut, address(_griefer), "");
        } else {
            fraxswapPair.swap(amountOut, 0, address(_griefer), "");
        }     
        vm.stopPrank();
 }
```

**Recommended Mitigation:** This issue exists for every new Agent created in AgentFactory and it is not feasible for the owner to check fee amount on every pair so, consider creating a new FraxSwapPair with the desired fee in AgentFactory::createAgent itself to avoid these fee discrepancies.

Consider any design change which will prevent arbitrary users to set fee.


### [M-2] Attacker can DOS liquidity migration in LiquidityManager.sol

**Description:** The vulnerability arises because the function relies on the raw ERC20 balance of the currency token held by the contract to calculate the amount of agent token liquidity to add. A malicious attacker can exploit this by injecting extra currency tokens into the LiquidityManager, artificially inflating the calculated liquidity amount and causing the function to fail.

**Background:** 

`LiquidityManager::moveLiquidity()`:
1. This function is responsible for migrating liquidity from the BootstrapPool to a FraxswapPair (a decentralized exchange pair).
2. It calculates the amount of agent tokens to add as liquidity based on the currency token balance held by the contract.

`Vulnerability`:
1. The function uses the raw ERC20 balance of the currency token to determine the liquidity amount:

```solidity
uint256 currencyAmount = currencyToken.balanceOf(address(this));
uint256 liquidityAmount = (currencyAmount * 1e18) / price;
```
2. This calculation assumes that the only currency tokens in the contract are those from the BootstrapPool.
3. However, an attacker can directly transfer extra currency tokens into the LiquidityManager, artificially inflating the currencyAmount and, consequently, the liquidityAmount.

`Exploit`:
1. When the inflated liquidityAmount is used to transfer agent tokens to the FraxswapPair, the contract may not have enough agent tokens to cover the transfer.
2. This results in an ERC20InsufficientBalance error, causing the function to revert and preventing liquidity migration.


**Impact:** Protocol Disruption: The migration process is a core functionality for moving liquidity from the bootstrap pool to the Fraxswap pair.
User Harm: Disrupted liquidity migration can result in poor market pricing or loss of confidence, potentially causing users to incur losses when trading or exiting positions.
Denial-of-Service (DoS): An attacker can exploit this vulnerability to prevent liquidity migration, indirectly impacting the protocol’s ability to function as intended.

**Recommended Mitigation:** Calculate the liquidity amount based on the currency amount that was received by the bootstrap pool. Proposed fix:
```js
uint256 currencyAmount = _reserveCurrencyToken; 
```
That way the attack wouldn’t work, and there’s also no risk of tokens getting left in the manager contract as you transfer them to the agent either way:
```js
        agentToken.safeTransfer(address(agent), agentToken.balanceOf(address(this)));
        currencyToken.safeTransfer(address(agent), currencyToken.balanceOf(address(this)));
```