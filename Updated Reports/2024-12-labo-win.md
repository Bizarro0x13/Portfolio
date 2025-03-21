### [H-1] Loss of User Funds in VirtualToken's `cashIn` function due to incorrect amount Minting

**Description:** In virtualToken contract `cashIn` function uses msg.value instead of amount for minting tokens when dealing with ERC20 tokens. This causes users to lose their deposited ERC20 tokens as they receive 0 virtual tokens in return.

**Impact:** 

**Proof of Concept:** The root cause is the incorrect usage of msg.value in the minting logic. While the function correctly handles the token transfer with the amount parameter, it incorrectly uses msg.value for minting, which is probably 0 for ERC20 token transactions. They receive 0 virtual tokens in return (since msg.value is 0 for ERC20 transactions)

```js
function cashIn(uint256 amount) external payable onlyWhiteListed {
    if (underlyingToken == LaunchPadUtils.NATIVE_TOKEN) {
        require(msg.value == amount, "Invalid ETH amount");
    } else {
        _transferAssetFromUser(amount);
    }
    // @audit Critical: Using msg.value instead of amount
    _mint(msg.sender, msg.value);  // Will be 0 for ERC20 tokens
    emit CashIn(msg.sender, msg.value);
}
```

**Recommended Mitigation:** 

```js
function cashIn(uint256 amount) external payable onlyWhiteListed {
    if (underlyingToken == LaunchPadUtils.NATIVE_TOKEN) {
        require(msg.value == amount, "Invalid ETH amount");
    } else {
        _transferAssetFromUser(amount);
    }
    _mint(msg.sender, amount);  // Use amount instead of msg.value
}
```

### [H-2] LamboFactory can be permanently DoS-ed due to `createPair` call reversal

**Description:** `LamboFactory.createLaunchPad` deploys new token contract and immediately sets up a new Uniswap V2 pool by calling `createPair`. This can be frontrun by the attacker by setting up a pool for the next token to be deployed.

Contract addresses are deterministic and can be calculated in advance. That opens a possibility for the attacker to pre-calculate the address of the next LamboToken to be deployed. As can be seen below, LamboFactory uses `clone()` method from OpenZeppelin `Clones` library, which uses `CREATE` EMV opCode under the hood.

```js
function _deployLamboToken(string memory name, string memory tickname) internal returns (address quoteToken) {
        // Create a deterministic clone of the LamboToken implementation
>>>     quoteToken = Clones.clone(lamboTokenImplementation);
        // Initialize the cloned LamboToken
        LamboToken(quoteToken).initialize(name, tickname);
        emit TokenDeployed(quoteToken);
    }
```

`CREATE` opcode calculates new contract address based on factory contract address and nonce (number of deployed contracts the factory has previously deployed):

`The destination address is calculated as the rightmost 20 bytes (160 bits) of the Keccak-256 hash of the rlp encoding of the sender address followed by its nonce. That is: address = keccak256(rlp([senderaddress,sendernonce]))[12:]`

Hence an attacker can calculate the address of the next token to be deployed and directly call `UniswapV2Factory.createPair` which will result in a new liquidity pool being created BEFORE the token has been deployed.

Such state will lead all subsequent calls to `LamboFactory.createLaunchPad` to revert, because of the pair existence check in Uniswap code, without the possibility to fix that:

https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Factory.sol#L27

```js
function createPair(address tokenA, address tokenB) external returns (address pair) {
        require(tokenA != tokenB, 'UniswapV2: IDENTICAL_ADDRESSES');
        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
        require(token0 != address(0), 'UniswapV2: ZERO_ADDRESS');
>>>     require(getPair[token0][token1] == address(0), 'UniswapV2: PAIR_EXISTS'); // single check is sufficient
        // ... the rest of the code is ommitted ...
    }
```

**Recommended Mitigation:** Check pool existence using `IUniswapV2Factory.getPair()`.


### [H-3] Calculation for `directionMask` is incorrect.

**Description:** The `_getQuoteAndDirection` function's flawed logic can cause incorrect direction determinatioon in the UniswapV3 pool. The recommended mitigation ensures that the function dynamically identifies token0 and token1 and assigns the correct direction mask. This prevents potential financial losses and ensures accurate rebalancing.

**Impact:** The function `previewRebalance` is called off-chain, to calculate values that can be passed to the function `rebalance` when making a call for balancing the uniswapV3 vETH/WETH pool.

```js
function previewRebalance()
        public
        view
        returns (bool result, uint256 directionMask, uint256 amountIn, uint256 amountOut)
    {
        address tokenIn;
        address tokenOut;
        (tokenIn, tokenOut, amountIn) = _getTokenInOut();
        (amountOut, directionMask) = _getQuoteAndDirection(tokenIn, tokenOut, amountIn);
        result = amountOut > amountIn;
    }
```
The function `_getQuoteAndDirection` takes tokenIn, tokenOut & `amountIn` as parameter to output amountOut & directionMask.
directionMask is used to decide if the swap is `zero-for-one` or `one-for-zero` in OKX.
the `_getQuoteAndDirection` function assumes that `WETH` is always token1, which can lead to incorect direction determination in cases where WETH is actually token0. This is due to the fact that Uniswap sorts token0 and token1 lexicographically by their addresses, and not based on their logical roles.

```js
function _getQuoteAndDirection(
        address tokenIn,
        address tokenOut,
        uint256 amountIn
    ) internal view returns (uint256 amountOut, uint256 directionMask) {
        (amountOut, , , ) = IQuoter(quoter).quoteExactInputSingleWithPool(
            IQuoter.QuoteExactInputSingleWithPoolParams({
                tokenIn: tokenIn,
                tokenOut: tokenOut,
                amountIn: amountIn,
                fee: fee,
                pool: uniswapPool,
                sqrtPriceLimitX96: 0
            })
        );
        >> directionMask = (tokenIn == weth) ? _BUY_MASK : _SELL_MASK;
    }
```
Example: if the UniswapV3 pool has token0 as WETH(lower address value) and token1 as vETH(higher address value), and the pool has more vETH than WETH, the tokenIn will be WETH. However, because WETH is token0 in this case, the correct direction would be zero-for-one. The current logic mistakenly assumes WETH is token1, leading to an incorrect direction of one-for-zero.

For context, here is how the MASK is used in OKX:
1. MASK defined
    `uint256 private constant _ONE_FOR_ZERO_MASK = 1 << 255; // Mask for identifying if the swap is one-for-zero`
2. MASK used
   `let zeroForOne := eq(and(_pool, _ONE_FOR_ZERO_MASK), 0)`

**Recommended Mitigation:** Add the logic for considering if the tokenIn is token0 or token1.

```js
function _getQuoteAndDirection(
    address tokenIn,
    address tokenOut,
    uint256 amountIn
) internal view returns (uint256 amountOut, uint256 directionMask) {
    // Retrieve token0 and token1 from the Uniswap pool
    address token0 = IUniswapV3Pool(uniswapPool).token0();
    address token1 = IUniswapV3Pool(uniswapPool).token1();

    // Call the quoter to get the amountOut
    (amountOut, , , ) = IQuoter(quoter).quoteExactInputSingleWithPool(
        IQuoter.QuoteExactInputSingleWithPoolParams({
            tokenIn: tokenIn,
            tokenOut: tokenOut,
            amountIn: amountIn,
            fee: fee,
            pool: uniswapPool,
            sqrtPriceLimitX96: 0
        })
    );
    // Determine directionMask based on tokenIn position (token0 or token1)
    if (tokenIn == token0) {
        directionMask = _SELL_MASK; // Zero-for-one direction
    } else {
        directionMask = _BUY_MASK; // One-for-zero direction
    }
}
```

### [H-4] Anyone can call `LamboRebalanceeOnUniswap.sol::rebalance()` function with any arbitrary value, leading to rebalancing goal i.e.(1:1 peg) unsuccessful. 

**Description:** Anyone can call `LamboRebalannceOnUniswap.sol::rebalance()` function with any arbitrary value, leading to rebalancing goal i.e. (1:1 peg) unsuccessful.

The parameters required in `rebalance()` function will are, `uint256 directionMask`, `uint256 amountIn`, `uint256 amountOut`. The ypical value should be - directionMask = `0` or `1<<255`
amountIn and amountOut obtained from `LamboRebalanceOnUniswap.sol::previewRebalance()` But since there is no check, to ensure the typical values of parameter in the function, this can cause the flashloan for wrong amount or flashloan reverting if directionMask is any other value apart from `0` or `1<<255`. amountIn and amountOut obtained from `LamboRebalanceOnUniwap.sol::previewRebalance()` 
But since there is no check, to ensure the typical values of parameter in the function, this can cause the flashLoan for wrong amount or flashloan reverting if directionMask is any other value apart from `0` or `1<<255`.
If flashloan of wrong amount occurs it means the pool will be unbalanced again with different value instead of balancing.

**Proof of Concept:** By pasting the following code in `RebalanceTest.t.sol`, we can see that `after_uniswapPoolWETHBalance:2` and `after_uniswapPoolVETHBalance:2` are much distant.

The test does the following - 

1. Do the usual rebalacing operation by executing `rebalance()`, by providing parameter from `previewRebalance()` and legit `directionMask`.
2. After snapshot revert, it calls the `rebalance()` function from an unauthorised user with an arbitrary value.
3. In the console log we can see, that the rebalance with typical parameters does the balancing goal of nearly 1:1

**Recommended Mitigation:** Check the parameter of `rebalance()` function whether they are legit or not, i.e. as per flashloan requirement.


### [M-1] Sincel the cost of launcing a new pool is minila, an attacker can maliciously consime `VirtualTokens`

**Description:** When launching a new pool, the factory contract needs to call the `takeLoan` function to intervene with virtual liquidity. The amount that can be borrowed is limiter to 300 ether per block.
https://github.com/code-423n4/2024-12-lambowin/blob/874fafc7b27042c59bdd765073f5e412a3b79192/src/VirtualToken.sol#L93

```js
 function takeLoan(address to, uint256 amount) external payable onlyValidFactory {
        if (block.number > lastLoanBlock) {
            lastLoanBlock = block.number;
            loanedAmountThisBlock = 0;
        }
        require(loanedAmountThisBlock + amount <= MAX_LOAN_PER_BLOCK, "Loan limit per block exceeded");
        loanedAmountThisBlock += amount;
        _mint(to, amount);
        _increaseDebt(to, amount);
        emit LoanTaken(to, amount);
```

However, when launching a new pool, the amount of virtual liquidity is controlled by users, and the minimum cost of launch a new pool is very low, requiring only gas fee and a small buy-in-fee. This allows attrackers to launch malicious new pools in each block, consuming the borrowing limit, which prevents legitimate users from launching new pools.

**Proof of Concept:**
1. User1 and User2  submit transactions to launch pools with virtual liquidity of 10 ether and 20 ether, respectively.
2. An attacker submits a transaction to launch a pool with 300 ether of virtual liquidity and offers a higher gas fee, causing their transaction to be prioritized and included in the block.
3. Due to the `takeLoan` debt limit, the transactions of User1 and User2 revert.
4. The attacker can repeat this attack in the next block.

```js
function test_createLaunchPool1() public {
        (address quoteToken, address pool, uint256 amountYOut) = lamboRouter.createLaunchPadAndInitialBuy{
            value: 10
        }(address(factory), "LamboToken", "LAMBO", 300 ether, 10);
    }
```
Running the above test can prove that an attacker only needs to consume the gas fee + 10 wei to exhaust the entire block's virtual liquidity available for creating new pools.

**Recommended Mitigation:** 
1. Charge a launch fee for new pools to increase attack costs.
2. Limit the maximum virtual liquidity a user can consume per transaction.


### [M-2] `LamboRebalanceOnUniswap::_getTokenInOut` formula used to compute rebalacing amount is wrong for a UniV3 pool

**Description:** The formula implemeted assumes that the pool is based on a constant sum AMM formula `(x+y = k)` , and also eludes the fact that reserves in a UniV3 pool do not directly relate to the price because of the 1-sided ticks liquidity.

This make the function imprecise at best, and highly imprecise when liquidity is deposited in distant ticks, with no risk involved for actors depositing in those ticks.

**Vulnerability details:** 
The `previewRebalance` function has been developed to output all the necessary input parameters required to call the `rebalance` function, which goal is to swap tokens in order to keep the peg of the virtual token in comparison to its counterpart (e.g keep vETH/ETH prices = 1):

```js
File: src/rebalance/LamboRebalanceOnUniwap.sol
128:     function _getTokenBalances() internal view returns (uint256 wethBalance, uint256 vethBalance) {
129:         wethBalance = IERC20(weth).balanceOf(uniswapPool);         <<❌(1) this does not represent the active tick
130:         vethBalance = IERC20(veth).balanceOf(uniswapPool);
131:     }
132: 
133:     function _getTokenInOut() internal view returns (address tokenIn, address tokenOut, uint256 amountIn) {
134:         (uint256 wethBalance, uint256 vethBalance) = _getTokenBalances();
135:         uint256 targetBalance = (wethBalance + vethBalance) / 2;    <<❌(2) wrong formula
136:
137:         if (vethBalance > targetBalance) {
138:             amountIn = vethBalance - targetBalance;
139:             tokenIn = weth;
140:             tokenOut = veth;
141:         } else {
142:             amountIn = wethBalance - targetBalance;
143:             tokenIn = veth;
144:             tokenOut = weth;
145:         }
146: 
147:         require(amountIn > 0, "amountIn must be greater than zero");
148:     }
149: 
```

The implemented formula is incorrect, as it will not rebalance the pool for 2 reasons:

1. In Uniswap V3, LPs can deposit tokens in any ticks they want, even though those ticks are not active and do not participate to the actual price. But those tokens will be held by the pool, and thus be measured by `_getTokenBalances`
2. The formula used to compute the targetBalance is incorrect because of how works the constant product formula `x*y=k`

Regarding (2), consider this situation: WETH balance: 1000 vETH balance: 900 `targetBalance = (1000 + 900) / 2 = 950` `amountIn = 1000 - 950 = 50 (vETH)`
Swapping 50 vETH into the pool will not return 50 WETH because of the inherent slippage of the constant product formula.
Now, add to this bullet (1), and the measured balance will be wrong anyway because of the liquidity deposited in inactive ticks, making the result even more shifted from the optimal result.

**Impact:** 
The function is not performing as intended, leading to invalid results which complicates the computation of rebalancing amounts necessary to maintain the peg.
Since this function is important to maintain the health of vETH as it has access to on-chain values, allowing precise rebalacing, failing to devise and implement a relaible solution for rebalancing before launch could result in significant issues.

**Recommended Mitigation:** Reconsider the computations of rebalancing amounts for a more precise one if keeping a 1:1 peg is important.
You might want to get inspiration from `USSDRebalancer::rebalance()`.

### [M-3] `sellQuote` and `buyQuote` are missing deadline check in `LamboVEthRouter`.

**Description:** https://github.com/code-423n4/2024-12-lambowin/blob/main/src/LamboVEthRouter.sol#L102-L102

https://github.com/code-423n4/2024-12-lambowin/blob/main/src/LamboVEthRouter.sol#L148-L148

`sellQuote` and `buyQuote` are missing deadline check in `LamboVEthRouter`.
Because of that, transactions can still be stuck in the mempool and be executed a long time after the transaction is initially called. During this time, the price in the Uniswap pool can change. In this case, the slippage parameters can become outdated and the swap will become vulnerable to sandwich attacks.

**Vulnerability details:** The protocol has made the choice to develop its own router to swap tokens for users, which imply calling the low level
`UniswapV2Pair::swap` function:
```js
// this low-level function should be called from a contract which performs important safety checks
function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external lock {
	require(amount0Out > 0 || amount1Out > 0, 'UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT');
	(uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
	require(amount0Out < _reserve0 && amount1Out < _reserve1, 'UniswapV2: INSUFFICIENT_LIQUIDITY');
```
As the comment indicates, this function require important safety checks to be performed.
A good example of safe implementation of such call can be found in the `UniswapV2Router02::swapExactTokensForTokens` function:

```js
function swapExactTokensForTokens(
	uint amountIn,
	uint amountOutMin,
	address[] calldata path,
	address to,
	uint deadline
) external virtual override ensure(deadline) returns (uint[] memory amounts) {
	amounts = UniswapV2Library.getAmountsOut(factory, amountIn, path);
	require(amounts[amounts.length - 1] >= amountOutMin, 'UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT');
	TransferHelper.safeTransferFrom(
		path[0], msg.sender, UniswapV2Library.pairFor(factory, path[0], path[1]), amounts[0]
	);
	_swap(amounts, path, to);
}
```
As we can see, 2 safety parameters are present here: `amountOutMin` and `deadline`.
Now, if we look at `SellQuote` (`buyQuote` having the same issue):
```js
File: src/LamboVEthRouter.sol
148:     function _buyQuote(address quoteToken, uint256 amountXIn, uint256 minReturn) internal returns (uint256 amountYOut) {
149:         require(msg.value >= amountXIn, "Insufficient msg.value");
150: 
...:
...:       //* ---------- some code ---------- *//
...:
168:         require(amountYOut >= minReturn, "Insufficient output amount");
```
We can see that no `deadline` parameter is present.

**Impact:** The transaction can still be stuck in the mempool and be executed a long time after the transaction is initially called. During this time, the price in the Uniswap pool can change. In this case, the slippage parameters can become outdated and the swap will become vulnerable to sandwich attacks.

**Recommended Mitigation:** Add a deadline parameter.

### [M-5] Incorrect Struct Field and Hardcoded `sqrtPriceLimitX96` in `_fgetQuoteAndDirection`

**Description:** The absence of a properly set `sqrtPriceLimit96` allows swaps to execute at pricefar beyond expected limits, exposing the contract to unfavourable trade outcomes. The function is also likely to fail at runtime due to mismatch in struct field names (`amountIn` instead of `amount`).

**Proof of Concept:** In the `getQuoteAndDirection` function, the `amountIn` parameter is incorrectly used for constructing the `QuoteExactInputSingleWithPoolParams` struct. Additionally, the sqrtPriceLimitX96 behavior in price-constrained swaps.

```js
IQuoter.QuoteExactInputSingleWithPoolParams({
                tokenIn: tokenIn,
                tokenOut: tokenOut,
                amountIn: amountIn, // Incorrect struct field usage
                fee: fee,
                pool: uniswapPool, 
                sqrtPriceLimitX96: 0 // No price constraint is applied
            })
```
The `QuoteExactOutputSingleWithPoolParams` struct implementation:

```js
struct QuoteExactOutputSingleWithPoolParams {
        address tokenIn;
        address tokenOut;
        uint256 amount;
        uint24 fee;
        address pool;
        uint160 sqrtPriceLimitX96;
    }
```

**Recommended Mitigation:** Update `_getQuoteAndDirection` to correctly reference the struct fields and provide configurable `sqrtPriceLimitX96` if needed:

```js
(amountOut, , , ) = IQuoter(quoter).quoteExactInputSingleWithPool(
    IQuoter.QuoteExactInputSingleWithPoolParams({
        tokenIn: tokenIn,
        tokenOut: tokenOut,
        amount: amountIn, // Correct field usage
        fee: fee,
        pool: uniswapPool,
        sqrtPriceLimitX96: sqrtPriceLimitX96 // Example of allowing no limit
    })
);
```

### [M-6] Attacker can capture `VETH-WETH` depeg profits through a malicious pool, rendering rebalancer useless if VETH Price > WETH Price

**Description:** https://github.com/code-423n4/2024-12-lambowin/blob/main/src/rebalance/LamboRebalanceOnUniwap.sol#L76

https://github.com/code-423n4/2024-12-lambowin/blob/main/src/rebalance/LamboRebalanceOnUniwap.sol#L109-L114

`LamboRebalanceOnUniswap::rebalance` accepts a `directionMask` argument, an arbitrary uint256 mask. It "OR" this mask with uniswapPool and passes it to the OKX router to perform `zeroForone` or `oneForZero` swaps (using the MSB).

```js
        function rebalance(uint256 directionMask, uint256 amountIn, uint256 amountOut) external nonReentrant {
        uint256 balanceBefore = IERC20(weth).balanceOf(address(this));
        bytes memory data = abi.encode(directionMask, amountIn, amountOut);
        IMorpho(morphoVault).flashLoan(weth, amountIn, data);
        uint256 balanceAfter = IERC20(weth).balanceOf(address(this));
        uint256 profit = balanceAfter - balanceBefore;
        require(profit > 0, "No profit made");
    }
```
However, this lets an attacker find a pool address with a malicious token. Attacker needs to find a pool with a malicious coin(dicussed in PoC) so that given:
```sh
malicious_mask = malicious_pool_address & (~uniswapPool)
```
we will have:
```sh
malicious_mask | uniswapPool = maliciousPool
```
Which allows attacker to insert the malicious pool here by passing `malicious_mask` to `rebalance` function:

```js
// given our malicious mask, _v3Pool is the desired pool
uint256 _v3pool = uint256(uint160(uniswapPool)) | (directionMask);
uint256[] memory pools = new uint256[](1); pools[0] = _v3pool;
```

Later, the first condition is not met since the `directionMask` is not equal to (1<<255). The code goes to the second condition:

```js
        if (directionMask == _BUY_MASK) {
            _executeBuy(amountIn, pools);
```

Second condition:
```js
        else {
        	_executeSell(amountIn, pools);
    	}
```

`_executeSell` calls `OKXROuter` with newly minted VETH (which we received at a discount, if VETH is priced higher than WETH). It then sends VETH to the malicious pool and receives malicious tokens + flash-loaned amount + 1 wei as the profit. The fact that its possible to replace the uniswapPool with our desired malicious pool opens up an attack path which if carefully executed, gives attacker the opportunity to profit from WETH-VETH depeg, leaving Lambo.win no profits at all.

The only difficult part of this attack is that attacker needs to deploy the malicious coin (which is paired with VETH) at a specific address so that their v3 pair address satisfies this condition:

```sh
    malicious_mask = malicious_pool_address & (~uniswapPool)
    malicious_mask | uniswapPool = maliciousPool
```
Basically the malicious_pool_address must have at leeast the same bits as the uniswap so that we can use a malicious mask on it.
Finding an address to satisfy this is hard(but not impossible, given current hardware advancements). For the sake of the PoC to be runnable, I have assumed the address of `uniswapPool` is 0x0000000000000000000000000000000000000093, so that we can find a malicious pool address and token easily. The actual difficulty of this attack depends on the actual address of WETH-VETH pool; however, I have used a simpler address, just to show that the attack is possible, given enough time.

After attacker deployed the right contracts, he can use them to profit from WETH-VETH depeges forever (unless new rebalancer is created).

**Recommended Mitigation:** Make sure that `directionMask` is either `(1 << 255)` or `0`.

### [M-7] Rebalance profit requirement prevents maintaining VETH?WETH peg

**Description:** https://github.com/code-423n4/2024-12-lambowin/blob/b8b8b0b1d7c9733a7bd9536e027886adb78ff83a/src/rebalance/LamboRebalanceOnUniwap.sol#L62

The profit > 0 requirement in the rebalance function actively prevents the protocol from maintaining the VETH/WETH 1:1 peg during unprofitable market conditions, when profit is ZERO.

**Proof of Concept:** The protocol documentation and team's design goals that the RebalanceOnUniswap contract is specifically designed to maintain the VETH/WETH pool ratio at 1:1, intentionally accepting gas losses as a trade-off for improved price stability.
It is mentioned in the previous audit, In the sponsor's acknowledgement (from SlowMist audit, N12):

```sh
According to the project team, the RebalanceOnUniswap contract is designed to maintain the VETH/WETH pool ratio at 1:1 rather than for profit. Gas costs are intentionally omitted to increase rebalancing frequency, accepting gas losses as a trade-off for improved price stability.
```
```js
    function rebalance(uint256 directionMask, uint256 amountIn, uint256 amountOut) external nonReentrant {
	    uint256 balanceBefore = IERC20(weth).balanceOf(address(this));
	    bytes memory data = abi.encode(directionMask, amountIn, amountOut);
	    IMorpho(morphoVault).flashLoan(weth, amountIn, data);
	    uint256 balanceAfter = IERC20(weth).balanceOf(address(this));
	    uint256 profit = balanceAfter - balanceBefore;
	    require(profit > 0, "No profit made");
    }
```

The `require(profit > 0)` check means:
Rebalancing can only occur when profitable, in situations where rebalancing is needed but arbitrage profits are zero, this directly contradicts the protocol's stated design goal of accepting no profit to maintain the ratio 1:1.

An example scenario would be:
1. VETH/WETH ration deviates from 1:1
2. Rebalancing opportunity exists to restore the peg
3. Market conditions mean rebalancing would offer no profit but can still done
4. The profit check prevents rebalancing

**Recommended Mitigation:** Update the `require(profit > 0)` to `require(profit >= 0)`.

### [M-8] Users can prevent protocol from rebalancing for his gain and cause loss of funds for protocol and its users

**Description:** https://github.com/code-423n4/2024-12-lambowin/blame/874fafc7b27042c59bdd765073f5e412a3b79192/src/rebalance/LamboRebalanceOnUniwap.sol#L62

The protocol have a vETH token that aims to be pegged to the ETH so the raio of vETH -> ETH = 1:1. When depeg happens the protocol can mitigate that via rebalance function in `LamboRebalanceOnUniswap` that looks like this:

```js
    function rebalance(uint256 directionMask, uint256 amountIn, uint256 amountOut) external nonReentrant {
    uint256 balanceBefore = IERC20(weth).balanceOf(address(this));
    bytes memory data = abi.encode(directionMask, amountIn, amountOut);
    IMorpho(morphoVault).flashLoan(weth, amountIn, data);
    uint256 balanceAfter = IERC20(weth).balanceOf(address(this));
    uint256 profit = balanceAfter - balanceBefore;
    require(profit > 0, "No profit made");
}
```
This function is designed to rebalance ratio by taking a flashloan from `MorphoVault`, which will be used on `UniswapV3` to make a swap, while at the same time make sure there is a profit for the caller which later can be transfered via the `extractprofit` function in the same contract.

**Proof of Concept:** The issue here is that rebalance function can be frontrun by malicious user wo will make a swap directly to the pool which will make the disbalance in ratio even greater and just enough for this check to revert:

```sh
    require(profit > 0, "No profit made");
```
Which will make the whole function to revert and `rebalance` will not happen.

**PoC:**
1. User call the `rebalance` function to restore vETH-ETH ratio
2. Malicious user sees the tx in mempool and front-runs it(for example to swap WETH to vETH which will inflate value of vETH)
3. Then the `rebalance` function is executed and flash loan is taken and going for swap.
4. The protocol receives less vETH than it should 
5. Not enough WETH is deposited so the `profit <= 0` which will make `rebalance` revert
6. Maicious user can furthermore use the vETH he got to swap and exploit the other Lambo tokens and inflate other pools for its gain via `LamboVETHRouter` as its expected ration in buyQuote is 1:1 for vETH/ETH (also by this comment here), by swapping his gained vETH for desired `targetToken`.

In the end, the attacker successfully DoS the rebalance function and since the rebalance is designed to ignore gas costs to increase rebalancing frequency, accepting gas losses as a trade-off for improved price stability. This will result in no profit but even a net loss of funds for the protocol and other users that are in the pool where one pair of tokens is inflated vETH. This will motivate malicious users to keep doinf these attacks and stop the protocol from rebalancing as long as it is profitable for him.

**Recommended Mitigation:** There is no easy solution but, one solution is to use previewRebalance in the rebalance function and compare the returned values and profitability with inputted amounts by the user so even ifd the attacker front-runs it, the function reverts even before flashloan is taken. That can save additional gas cost and it will make each time more expensive and unprofitable for the attacker to perform that attack. Also to include a mechanism that will account for the possibility of inflated vETH values in `LamboVETHRouter`.


### [M-9] Rebalance will be completely dossed if OKX comission rate goes beyond the fee limits

**Description:** https://github.com/code-423n4/2024-12-lambowin/blob/874fafc7b27042c59bdd765073f5e412a3b79192/src/rebalance/LamboRebalanceOnUniwap.sol#L89-L114

Rebalancing interacts with OKXRouter to swap weth for tokens in certain pools and vice versa. But OKXRouter may charge commissions on both the from token and to token, which reduces potential profit to be made from the rebalance operation, if high enough causes there to be no profit made, which will cause the operation to fail, or in extreme cases, if the commision is set beyond its expected limits, cause a permanent dos of the function.

**Impact:** 

**Proof of Concept:** `rebalance` calls morpho `flashloan` which calls the `onMorphoFlashLoan` hook.

```js
    function rebalance(uint256 directionMask, uint256 amountIn, uint256 amountOut) external nonReentrant {
        uint256 balanceBefore = IERC20(weth).balanceOf(address(this));
        bytes memory data = abi.encode(directionMask, amountIn, amountOut);
>>      IMorpho(morphoVault).flashLoan(weth, amountIn, data);
        uint256 balanceAfter = IERC20(weth).balanceOf(address(this));
        uint256 profit = balanceAfter - balanceBefore;
        require(profit > 0, "No profit made");
    }
```
`onMorphoFlashLoan`, depending on the set direction executes buy or sell transaction using OKXRouter.

```js
    function onMorphoFlashLoan(uint256 assets, bytes calldata data) external {
        require(msg.sender == address(morphoVault), "Caller is not morphoVault");
        (uint256 directionMask, uint256 amountIn, uint256 amountOut) = abi.decode(data, (uint256, uint256, uint256));
        require(amountIn == assets, "Amount in does not match assets");
        uint256 _v3pool = uint256(uint160(uniswapPool)) | (directionMask);
        uint256[] memory pools = new uint256[](1);
        pools[0] = _v3pool;
        if (directionMask == _BUY_MASK) {
>>          _executeBuy(amountIn, pools);
        } else {
>>          _executeSell(amountIn, pools);
        }
        require(IERC20(weth).approve(address(morphoVault), assets), "Approve failed");
    }
```
OKXRouter, for its transactions charges a fee from both the from and to token. This is important as according to the readme, issues from external integration enabling fees while affecting the protocol are in scope.

```js
    function _executeBuy(uint256 amountIn, uint256[] memory pools) internal {
        uint256 initialBalance = address(this).balance;
        // Execute buy
        require(IERC20(weth).approve(address(OKXTokenApprove), amountIn), "Approve failed");
>>      uint256 uniswapV3AmountOut = IDexRouter(OKXRouter).uniswapV3SwapTo(
            uint256(uint160(address(this))),
            amountIn,
            0,
            pools
        );
        VirtualToken(veth).cashOut(uniswapV3AmountOut);
        // SlowMist [N11]
        uint256 newBalance = address(this).balance - initialBalance;
        if (newBalance > 0) {
            IWETH(weth).deposit{value: newBalance}();
        }
    }
    function _executeSell(uint256 amountIn, uint256[] memory pools) internal {
        IWETH(weth).withdraw(amountIn);
        VirtualToken(veth).cashIn{value: amountIn}(amountIn);
>>      require(IERC20(veth).approve(address(OKXTokenApprove), amountIn), "Approve failed");
        IDexRouter(OKXRouter).uniswapV3SwapTo(uint256(uint160(address(this))), amountIn, 0, pools);
    }
```
From the router's uniswapV3SwapTo function, we can see that a commission in taken from the from token and from the to token after swap. The presence of these fees, for the first part reduces the potential profit that the protocol stands to make from the flashLoan, leading to a lass of posisitve yield. Worse still is if the commision is high enough, no profit will be made, causing the rebalance function to revert due to the requirement that profit is made during every rebalance operation.

```js
    function _uniswapV3SwapTo(
        address payer,
        uint256 receiver,
        uint256 amount,
        uint256 minReturn,
        uint256[] calldata pools
    ) internal returns (uint256 returnAmount) {
        CommissionInfo memory commissionInfo = _getCommissionInfo();
        (
            address middleReceiver,
            uint256 balanceBefore
>>      ) = _doCommissionFromToken(
                commissionInfo,
                address(uint160(receiver)),
                amount
            );
        (uint256 swappedAmount, ) = _uniswapV3Swap(
            payer,
            payable(middleReceiver),
            amount,
            minReturn,
            pools
        );
>>      uint256 commissionAmount = _doCommissionToToken(
            commissionInfo,
            address(uint160(receiver)),
            balanceBefore
        );
        return swappedAmount - commissionAmount;
    }
```
And finally, if the commission rate exceeds its preset commissionRateLimit either for the fromToken or toToken, will also cause the function to also revert, dossing rebalancing

```js
    function _doCommissionFromToken(
        CommissionInfo memory commissionInfo,
        address receiver,
        uint256 inputAmount
    ) internal returns (address, uint256) {
//...
            let rate := mload(add(commissionInfo, 0x40))
>>          if gt(rate, commissionRateLimit) {
                _revertWithReason(
                    0x0000001b6572726f7220636f6d6d697373696f6e2072617465206c696d697400,
                    0x5f
                ) //"error commission rate limit"
            }
//...


    function _doCommissionToToken(
        CommissionInfo memory commissionInfo,
        address receiver,
        uint256 balanceBefore
    ) internal returns (uint256 amount) {
        if (!commissionInfo.isToTokenCommission) {
            return 0;
        }
//...
            let rate := mload(add(commissionInfo, 0x40))
>>           if gt(rate, commissionRateLimit) {
                _revertWithReason(
                    0x0000001b6572726f7220636f6d6d697373696f6e2072617465206c696d697400,
                    0x5f
                ) //"error commission rate limit"
            }
//...
```