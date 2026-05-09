# Uniswap V2 Router

### Periphery Contracts <a href="#periphery-contracts" id="periphery-contracts"></a>

The periphery contracts are the API layer for Uniswap. They are intended for external calls from other contracts or decentralized applications. You can call the core contracts directly, but doing so is more complex, and a mistake can lose value. The core contracts only contain the checks needed to protect themselves from being cheated; they do not perform all of the caller-oriented sanity checks that application developers usually need. The contracts are kept in the periphery so they can be replaced or upgraded when necessary.

#### UniswapV2Router01.sol <a href="#uniswapv2router01" id="uniswapv2router01"></a>

[This contract](https://github.com/Uniswap/uniswap-v2-periphery/blob/master/contracts/UniswapV2Router01.sol) has known issues and [should no longer be used](https://docs.uniswap.org/protocol/V2/reference/smart-contracts/router-01/). Fortunately, periphery contracts are stateless and do not own assets, so they are easy to deprecate. Use `UniswapV2Router02` instead.

#### UniswapV2Router02.sol <a href="#uniswapv2router02" id="uniswapv2router02"></a>

In most cases, this is [the contract](https://github.com/Uniswap/uniswap-v2-periphery/blob/master/contracts/UniswapV2Router02.sol) you interact with when using Uniswap V2. Usage instructions are available in the [Router02 reference](https://docs.uniswap.org/protocol/V2/reference/smart-contracts/router-02/).

```solidity
pragma solidity =0.6.6;

import '@uniswap/v2-core/contracts/interfaces/IUniswapV2Factory.sol';
import '@uniswap/lib/contracts/libraries/TransferHelper.sol';

import './interfaces/IUniswapV2Router02.sol';
import './libraries/UniswapV2Library.sol';
import './libraries/SafeMath.sol';
import './interfaces/IERC20.sol';
import './interfaces/IWETH.sol';
```

Most of these imports are familiar or self-explanatory. The notable exception is `IWETH.sol`. Uniswap V2 can swap arbitrary ERC-20 token pairs, but Ether (ETH) itself is not an ERC-20 token. ETH predates the token standard and has its own transfer mechanics. To use ETH inside contracts that expect ERC-20 tokens, the ecosystem uses the [Wrapped Ether (WETH)](https://weth.io/) contract. When you send ETH to the WETH contract, it mints the same amount of WETH for you; when you burn WETH, you receive the corresponding amount of ETH back.

```solidity
contract UniswapV2Router02 is IUniswapV2Router02 {
    using SafeMath for uint;

    address public immutable override factory;
    address public immutable override WETH;
```

The router needs to know which factory to use and which Wrapped Ether contract should be used for trades involving ETH. These variables are [immutable](https://docs.soliditylang.org/en/v0.8.3/contracts.html#constant-and-immutable-state-variables), which means they can only be set in the constructor. This lets users rely on the fact that nobody can later change them to point to a risky or malicious contract.

```solidity
    modifier ensure(uint deadline) {
        require(deadline >= block.timestamp, 'UniswapV2Router: EXPIRED');
        _;
    }
```

This modifier ensures that time-bounded transactions, such as "perform X before time Y if possible," cannot execute after their deadline.

```solidity
    constructor(address _factory, address _WETH) public {
        factory = _factory;
        WETH = _WETH;
    }
```

The constructor only sets the immutable state variables.

```solidity
    receive() external payable {
        assert(msg.sender == WETH); // only accept ETH via fallback from the WETH contract
    }
```

This function is needed when the router converts WETH back into ETH. Only the WETH contract configured for this router is allowed to send ETH through this receive hook.

**Add Liquidity**

These functions add tokens to a pair, increasing the liquidity pool.

```solidity
    // **** ADD LIQUIDITY ****
    function _addLiquidity(
```

This function calculates how much of token A and token B should be deposited into the pair.

```solidity
        address tokenA,
        address tokenB,
```

These are the addresses of the ERC-20 token contracts.

```solidity
        uint amountADesired,
        uint amountBDesired,
```

These are the token amounts the liquidity provider would like to deposit. They are also the maximum amounts of A and B that may be deposited.

```solidity
        uint amountAMin,
        uint amountBMin
```

These are the minimum acceptable deposit amounts. If the transaction cannot be completed with amounts at or above these thresholds, it reverts. If this protection is not needed, they can be set to zero.

Liquidity providers specify minimum amounts to constrain the transaction to an exchange rate close to the current rate. If the rate moves too far, the underlying value relationship may have changed, and the provider may need to decide manually what to do.

For example, suppose the exchange rate is one-to-one and the liquidity provider specifies these values:

| Parameter      | Value |
| -------------- | ---: |
| amountADesired | 1000 |
| amountBDesired | 1000 |
| amountAMin     |  900 |
| amountBMin     |  800 |

As long as the exchange rate stays between 0.9 and 1.25, the transaction can proceed. If the rate moves outside that range, the transaction is cancelled.

This protection is needed because transactions do not execute immediately. After a transaction is submitted, execution only happens when a block producer includes it in a block. If the gas price is too low, the transaction may remain pending until another transaction with the same nonce and a higher gas price replaces it. The user cannot control what happens between submission and inclusion in a block.

```solidity
    ) internal virtual returns (uint amountA, uint amountB) {
```

The function returns the amounts the liquidity provider should deposit, using the same ratio as the current reserves.

```solidity
        // create the pair if it doesn't exist yet
        if (IUniswapV2Factory(factory).getPair(tokenA, tokenB) == address(0)) {
            IUniswapV2Factory(factory).createPair(tokenA, tokenB);
        }
```

If the pair for these two tokens does not exist yet, create it.

```solidity
        (uint reserveA, uint reserveB) = UniswapV2Library.getReserves(factory, tokenA, tokenB);
```

Fetch the pair's current reserves.

```solidity
        if (reserveA == 0 && reserveB == 0) {
            (amountA, amountB) = (amountADesired, amountBDesired);
```

If the reserves are empty, this is a new pair. The deposited amounts should exactly match what the liquidity provider wanted to provide.

```solidity
        } else {
            uint amountBOptimal = UniswapV2Library.quote(amountADesired, reserveA, reserveB);
```

When we need to compute the corresponding optimal amount, we use [this function](https://github.com/Uniswap/uniswap-v2-periphery/blob/master/contracts/libraries/UniswapV2Library.sol#L35). The goal is to match the current reserve ratio.

```solidity
            if (amountBOptimal <= amountBDesired) {
                require(amountBOptimal >= amountBMin, 'UniswapV2Router: INSUFFICIENT_B_AMOUNT');
                (amountA, amountB) = (amountADesired, amountBOptimal);
```

If the optimal amount `amountBOptimal` is smaller than the amount the liquidity provider wanted to deposit, token B is currently worth more than the provider assumed, so a smaller amount of B is needed.

```solidity
            } else {
                uint amountAOptimal = UniswapV2Library.quote(amountBDesired, reserveB, reserveA);
                assert(amountAOptimal <= amountADesired);
                require(amountAOptimal >= amountAMin, 'UniswapV2Router: INSUFFICIENT_A_AMOUNT');
                (amountA, amountB) = (amountAOptimal, amountBDesired);
```

If the optimal amount of token B is larger than the desired amount of token B, token B is currently worth less than the liquidity provider estimated, so more B would be needed. However, the desired amount is also the maximum, which means we cannot deposit more B. The alternative is to compute the optimal amount of token A for the desired amount of token B.

Putting these values together gives the chart described below. Assume you are trying to deposit 1000 A tokens (blue line) and 1000 B tokens (red line). The x-axis is the exchange rate, A/B. If `x = 1`, the tokens have equal value, and you can deposit 1000 A tokens and 1000 B tokens. If `x = 2`, A is worth twice as much as B, so each A token can be exchanged for two B tokens. In that case, you can deposit 1000 B tokens but only 500 A tokens. If `x = 0.5`, the situation is reversed: you can deposit 1000 A tokens or 500 B tokens.

> The original tutorial includes a liquidity-deposit flow diagram here, but the image file is not present in the current notes repository.

```solidity
            }
        }
    }
```

You can deposit liquidity directly into the core contract by using [`UniswapV2Pair::mint`](https://github.com/Uniswap/uniswap-v2-core/blob/master/contracts/UniswapV2Pair.sol#L110), but the core contract only checks that the pair itself is not cheated. If the exchange rate changes between transaction submission and execution, you risk depositing at an unfavorable value. When you use the periphery contract, it calculates the amounts you should deposit and deposits them immediately in the same transaction, so the rate cannot change between calculation and deposit.

```solidity
    function addLiquidity(
        address tokenA,
        address tokenB,
        uint amountADesired,
        uint amountBDesired,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
```

This function can be called in a transaction to deposit liquidity. Most parameters are the same as in `_addLiquidity`, with two additions:

* `to` is the address that receives the newly minted liquidity tokens, which represent the liquidity provider's share of the pool.
* `deadline` is the transaction deadline.

```solidity
    ) external virtual override ensure(deadline) returns (uint amountA, uint amountB, uint liquidity) {
        (amountA, amountB) = _addLiquidity(tokenA, tokenB, amountADesired, amountBDesired, amountAMin, amountBMin);
        address pair = UniswapV2Library.pairFor(factory, tokenA, tokenB);
```

We calculate the actual deposited amounts and then find the liquidity pool account address. To save gas, we do not query the factory; instead we use the library function `pairFor` (see the library section below).

```solidity
        TransferHelper.safeTransferFrom(tokenA, msg.sender, pair, amountA);
        TransferHelper.safeTransferFrom(tokenB, msg.sender, pair, amountB);
```

Transfer the correct token amounts from the user's account to the pair.

```solidity
        liquidity = IUniswapV2Pair(pair).mint(to);
    }
```

In return, the pair mints liquidity tokens to the `to` address, representing partial ownership of the liquidity pool. The core contract's `mint` function observes how many extra tokens it now holds compared with the previous liquidity state, and mints liquidity tokens accordingly.

```solidity
    function addLiquidityETH(
        address token,
        uint amountTokenDesired,
```

There are a few differences when a liquidity provider wants to provide liquidity to a token/ETH pair. The contract handles wrapping ETH for the provider. The user does not need to specify how much ETH they want to deposit separately, because ETH can be sent directly with the transaction and read from `msg.value`.

```solidity
        uint amountTokenMin,
        uint amountETHMin,
        address to,
        uint deadline
    ) external virtual override payable ensure(deadline) returns (uint amountToken, uint amountETH, uint liquidity) {
        (amountToken, amountETH) = _addLiquidity(
            token,
            WETH,
            amountTokenDesired,
            msg.value,
            amountTokenMin,
            amountETHMin
        );
        address pair = UniswapV2Library.pairFor(factory, token, WETH);
        TransferHelper.safeTransferFrom(token, msg.sender, pair, amountToken);
        IWETH(WETH).deposit{value: amountETH}();
        assert(IWETH(WETH).transfer(pair, amountETH));
```

To deposit ETH into the pair, the router first wraps it into WETH and then transfers the WETH to the pair. Notice that the transfer is wrapped in an `assert`. If the transfer fails, the whole contract call fails, so the wrapping does not effectively persist.

```solidity
        liquidity = IUniswapV2Pair(pair).mint(to);
        // refund dust eth, if any
        if (msg.value > amountETH) TransferHelper.safeTransferETH(msg.sender, msg.value - amountETH);
    }
```

The user has already sent ETH to the router. If any extra ETH remains because the other token is worth less than the user assumed, the router refunds the excess.

**Remove Liquidity**

The following functions withdraw liquidity and return the underlying assets to the liquidity provider.

```solidity
    // **** REMOVE LIQUIDITY ****
    function removeLiquidity(
        address tokenA,
        address tokenB,
        uint liquidity,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    ) public virtual override ensure(deadline) returns (uint amountA, uint amountB) {
```

This is the simplest liquidity removal case. The liquidity provider agrees to accept at least a minimum amount of each token, and the transaction must complete before the deadline.

```solidity
        address pair = UniswapV2Library.pairFor(factory, tokenA, tokenB);
        IUniswapV2Pair(pair).transferFrom(msg.sender, pair, liquidity); // send liquidity to pair
        (uint amount0, uint amount1) = IUniswapV2Pair(pair).burn(to);
```

The core contract's `burn` function handles returning tokens to the user.

```solidity
        (address token0,) = UniswapV2Library.sortTokens(tokenA, tokenB);
```

When a function returns multiple values and we only care about some of them, this is how Solidity lets us retrieve only the values we need. From a gas perspective, this is cheaper than reading values that are never used.

```solidity
        (amountA, amountB) = tokenA == token0 ? (amount0, amount1) : (amount1, amount0);
```

The core contract returns amounts in token-sorted order. This line converts them back into the order expected by the caller, corresponding to `tokenA` and `tokenB`.

```solidity
        require(amountA >= amountAMin, 'UniswapV2Router: INSUFFICIENT_A_AMOUNT');
        require(amountB >= amountBMin, 'UniswapV2Router: INSUFFICIENT_B_AMOUNT');
    }
```

The token transfer can happen before the legality checks because, if a check fails, the transaction reverts and all state changes are rolled back.

```solidity
    function removeLiquidityETH(
        address token,
        uint liquidity,
        uint amountTokenMin,
        uint amountETHMin,
        address to,
        uint deadline
    ) public virtual override ensure(deadline) returns (uint amountToken, uint amountETH) {
        (amountToken, amountETH) = removeLiquidity(
            token,
            WETH,
            liquidity,
            amountTokenMin,
            amountETHMin,
            address(this),
            deadline
        );
        TransferHelper.safeTransfer(token, to, amountToken);
        IWETH(WETH).withdraw(amountETH);
        TransferHelper.safeTransferETH(to, amountETH);
    }
```

Removing ETH liquidity is almost the same. The difference is that the router first receives WETH, unwraps it back into ETH, and then sends the ETH to the liquidity provider.

```solidity
    function removeLiquidityWithPermit(
        address tokenA,
        address tokenB,
        uint liquidity,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline,
        bool approveMax, uint8 v, bytes32 r, bytes32 s
    ) external virtual override returns (uint amountA, uint amountB) {
        address pair = UniswapV2Library.pairFor(factory, tokenA, tokenB);
        uint value = approveMax ? uint(-1) : liquidity;
        IUniswapV2Pair(pair).permit(msg.sender, address(this), value, deadline, v, r, s);
        (amountA, amountB) = removeLiquidity(tokenA, tokenB, liquidity, amountAMin, amountBMin, to, deadline);
    }


    function removeLiquidityETHWithPermit(
        address token,
        uint liquidity,
        uint amountTokenMin,
        uint amountETHMin,
        address to,
        uint deadline,
        bool approveMax, uint8 v, bytes32 r, bytes32 s
    ) external virtual override returns (uint amountToken, uint amountETH) {
        address pair = UniswapV2Library.pairFor(factory, token, WETH);
        uint value = approveMax ? uint(-1) : liquidity;
        IUniswapV2Pair(pair).permit(msg.sender, address(this), value, deadline, v, r, s);
        (amountToken, amountETH) = removeLiquidityETH(token, liquidity, amountTokenMin, amountETHMin, to, deadline);
    }
```

These functions support meta-transaction-style removal by using the [permit mechanism](README.md#UniswapV2ERC20), allowing users to authorize liquidity-token transfers without sending a separate approval transaction first.

```solidity
    // **** REMOVE LIQUIDITY (supporting fee-on-transfer tokens) ****
    function removeLiquidityETHSupportingFeeOnTransferTokens(
        address token,
        uint liquidity,
        uint amountTokenMin,
        uint amountETHMin,
        address to,
        uint deadline
    ) public virtual override ensure(deadline) returns (uint amountETH) {
        (, amountETH) = removeLiquidity(
            token,
            WETH,
            liquidity,
            amountTokenMin,
            amountETHMin,
            address(this),
            deadline
        );
        TransferHelper.safeTransfer(token, to, IERC20(token).balanceOf(address(this)));
        IWETH(WETH).withdraw(amountETH);
        TransferHelper.safeTransferETH(to, amountETH);
    }
```

This function is for tokens that charge a fee during transfer or storage. When a token contract has such a fee, we cannot rely on `removeLiquidity` to tell us exactly how many tokens are withdrawable. Instead, the router withdraws first and then checks the token balance.

```solidity
    function removeLiquidityETHWithPermitSupportingFeeOnTransferTokens(
        address token,
        uint liquidity,
        uint amountTokenMin,
        uint amountETHMin,
        address to,
        uint deadline,
        bool approveMax, uint8 v, bytes32 r, bytes32 s
    ) external virtual override returns (uint amountETH) {
        address pair = UniswapV2Library.pairFor(factory, token, WETH);
        uint value = approveMax ? uint(-1) : liquidity;
        IUniswapV2Pair(pair).permit(msg.sender, address(this), value, deadline, v, r, s);
        amountETH = removeLiquidityETHSupportingFeeOnTransferTokens(
            token, liquidity, amountTokenMin, amountETHMin, to, deadline
        );
    }
```

This final function combines fee-on-transfer support with permit-based liquidity removal.

**Swaps**

```solidity
    // **** SWAP ****
    // requires the initial amount to have already been sent to the first pair
    function _swap(uint[] memory amounts, address[] memory path, address _to) internal virtual {
```

The externally exposed swap functions call this internal function to perform the actual swap sequence.

```solidity
        for (uint i; i < path.length - 1; i++) {
```

At the time of the original tutorial, there were [388,160 ERC-20 tokens](https://etherscan.io/tokens). If every token pair had its own pair contract, the number of pairs would exceed 150 billion. At that time, the entire chain had [only about 0.1% of that number of accounts](https://etherscan.io/chart/address). In practice, the swap functions support the concept of a path. A trader can swap A for B, B for C, and C for D, so a direct A-D pair is not required.

Prices across these markets tend to remain synchronized because unsynchronized prices create arbitrage opportunities. For example, suppose there are three tokens, A, B, and C, with one pair for each relationship.

1. Initial state.
2. The trader sells 24.695 A tokens and receives 25.305 B tokens.
3. The trader sells 24.695 B tokens for 25.305 C tokens, gaining roughly 0.61 B tokens.
4. The trader sells 24.695 C tokens for 25.305 A tokens, gaining roughly 0.61 C tokens. The trader also keeps the remaining 0.61 A tokens, because the final 25.305 A tokens exceed the original 24.695 A-token input.

| Step | A-B pair                     | B-C pair                     | A-C pair                     |
| -- | --------------------------- | --------------------------- | --------------------------- |
| 1  | A:1000 B:1050 A/B=1.05      | B:1000 C:1050 B/C=1.05      | A:1050 C:1000 C/A=1.05      |
| 2  | A:1024.695 B:1024.695 A/B=1 | B:1000 C:1050 B/C=1.05      | A:1050 C:1000 C/A=1.05      |
| 3  | A:1024.695 B:1024.695 A/B=1 | B:1024.695 C:1024.695 B/C=1 | A:1050 C:1000 C/A=1.05      |
| 4  | A:1024.695 B:1024.695 A/B=1 | B:1024.695 C:1024.695 B/C=1 | A:1024.695 C:1024.695 C/A=1 |

```solidity
            (address input, address output) = (path[i], path[i + 1]);
            (address token0,) = UniswapV2Library.sortTokens(input, output);
            uint amountOut = amounts[i + 1];
```

Get the pair currently being processed, sort the token addresses into the pair's expected order, and read the expected output amount.

```solidity
            (uint amount0Out, uint amount1Out) = input == token0 ? (uint(0), amountOut) : (amountOut, uint(0));
```

After computing the expected amount, arrange it in the order expected by the pair contract.

```solidity
            address to = i < path.length - 2 ? UniswapV2Library.pairFor(factory, output, path[i + 2]) : _to;
```

Is this the final swap in the path? If so, send the received output tokens to the destination address. If not, send them to the next pair contract.

```solidity
            IUniswapV2Pair(UniswapV2Library.pairFor(factory, input, output)).swap(
                amount0Out, amount1Out, to, new bytes(0)
            );
        }
    }
```

This is the actual pair call that swaps the tokens. We do not need a callback payload for this trade, so the data field is empty.

```solidity
    function swapExactTokensForTokens(
```

Traders call this function directly to swap tokens.

```solidity
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
```

This parameter contains ERC-20 contract addresses. As described above, it is an array because the swap may need to traverse multiple pairs to move from the input asset to the desired output asset.

In Solidity, function parameters can live in `memory` or `calldata`. If the function is an entry point to the contract, meaning it is called directly by a user transaction or by another contract, the parameter values can be read directly from the call data. If the function is called internally, as `_swap` is above, the parameters must be stored in `memory`. From the called contract's perspective, `calldata` is read-only.

For scalar types such as `uint` or `address`, the compiler can usually handle the storage-location choice for us. For arrays, because they require more storage and cost more gas, we need to specify which storage location to use.

```solidity
        address to,
        uint deadline
    ) external virtual override ensure(deadline) returns (uint[] memory amounts) {
```

Return values are always returned in memory.

```solidity
        amounts = UniswapV2Library.getAmountsOut(factory, amountIn, path);
        require(amounts[amounts.length - 1] >= amountOutMin, 'UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT');
```

Compute the token amounts to receive at each hop. If the final result is lower than the minimum the trader is willing to accept, revert the transaction.

```solidity
        TransferHelper.safeTransferFrom(
            path[0], msg.sender, UniswapV2Library.pairFor(factory, path[0], path[1]), amounts[0]
        );
        _swap(amounts, path, to);
    }
```

Finally, transfer the initial ERC-20 tokens to the first pair account and call `_swap`. All of this happens in a single transaction, so the pair can treat any unexpected token balance increase as part of this swap.

```solidity
    function swapTokensForExactTokens(
        uint amountOut,
        uint amountInMax,
        address[] calldata path,
        address to,
        uint deadline
    ) external virtual override ensure(deadline) returns (uint[] memory amounts) {
        amounts = UniswapV2Library.getAmountsIn(factory, amountOut, path);
        require(amounts[0] <= amountInMax, 'UniswapV2Router: EXCESSIVE_INPUT_AMOUNT');
        TransferHelper.safeTransferFrom(
            path[0], msg.sender, UniswapV2Library.pairFor(factory, path[0], path[1]), amounts[0]
        );
        _swap(amounts, path, to);
    }
```

The previous function, `swapExactTokensForTokens`, lets the trader specify the exact amount of input tokens and the minimum amount of output tokens they are willing to accept. This function reverses that framing: the trader specifies the exact output amount they want and the maximum input amount they are willing to pay.

In both cases, the trader must first approve this periphery contract to transfer the relevant input tokens.

```solidity
    function swapExactETHForTokens(uint amountOutMin, address[] calldata path, address to, uint deadline)
        external
        virtual
        override
        payable
        ensure(deadline)
        returns (uint[] memory amounts)
    {
        require(path[0] == WETH, 'UniswapV2Router: INVALID_PATH');
        amounts = UniswapV2Library.getAmountsOut(factory, msg.value, path);
        require(amounts[amounts.length - 1] >= amountOutMin, 'UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT');
        IWETH(WETH).deposit{value: amounts[0]}();
        assert(IWETH(WETH).transfer(UniswapV2Library.pairFor(factory, path[0], path[1]), amounts[0]));
        _swap(amounts, path, to);
    }


    function swapTokensForExactETH(uint amountOut, uint amountInMax, address[] calldata path, address to, uint deadline)
        external
        virtual
        override
        ensure(deadline)
        returns (uint[] memory amounts)
    {
        require(path[path.length - 1] == WETH, 'UniswapV2Router: INVALID_PATH');
        amounts = UniswapV2Library.getAmountsIn(factory, amountOut, path);
        require(amounts[0] <= amountInMax, 'UniswapV2Router: EXCESSIVE_INPUT_AMOUNT');
        TransferHelper.safeTransferFrom(
            path[0], msg.sender, UniswapV2Library.pairFor(factory, path[0], path[1]), amounts[0]
        );
        _swap(amounts, path, address(this));
        IWETH(WETH).withdraw(amounts[amounts.length - 1]);
        TransferHelper.safeTransferETH(to, amounts[amounts.length - 1]);
    }



    function swapExactTokensForETH(uint amountIn, uint amountOutMin, address[] calldata path, address to, uint deadline)
        external
        virtual
        override
        ensure(deadline)
        returns (uint[] memory amounts)
    {
        require(path[path.length - 1] == WETH, 'UniswapV2Router: INVALID_PATH');
        amounts = UniswapV2Library.getAmountsOut(factory, amountIn, path);
        require(amounts[amounts.length - 1] >= amountOutMin, 'UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT');
        TransferHelper.safeTransferFrom(
            path[0], msg.sender, UniswapV2Library.pairFor(factory, path[0], path[1]), amounts[0]
        );
        _swap(amounts, path, address(this));
        IWETH(WETH).withdraw(amounts[amounts.length - 1]);
        TransferHelper.safeTransferETH(to, amounts[amounts.length - 1]);
    }


    function swapETHForExactTokens(uint amountOut, address[] calldata path, address to, uint deadline)
        external
        virtual
        override
        payable
        ensure(deadline)
        returns (uint[] memory amounts)
    {
        require(path[0] == WETH, 'UniswapV2Router: INVALID_PATH');
        amounts = UniswapV2Library.getAmountsIn(factory, amountOut, path);
        require(amounts[0] <= msg.value, 'UniswapV2Router: EXCESSIVE_INPUT_AMOUNT');
        IWETH(WETH).deposit{value: amounts[0]}();
        assert(IWETH(WETH).transfer(UniswapV2Library.pairFor(factory, path[0], path[1]), amounts[0]));
        _swap(amounts, path, to);
        // refund dust eth, if any
        if (msg.value > amounts[0]) TransferHelper.safeTransferETH(msg.sender, msg.value - amounts[0]);
    }
```

These four variants all involve swaps between ETH and tokens. The difference is only whether the router receives ETH from the trader and wraps it into WETH, or receives WETH from the last swap in the path, unwraps it, and sends ETH to the trader.

```solidity
    // **** SWAP (supporting fee-on-transfer tokens) ****
    // requires the initial amount to have already been sent to the first pair
    function _swapSupportingFeeOnTransferTokens(address[] memory path, address _to) internal virtual {
```

This internal function is used for swaps involving tokens that charge transfer or storage fees. It addresses cases such as [this issue](https://github.com/Uniswap/interface/issues/835).

```solidity
        for (uint i; i < path.length - 1; i++) {
            (address input, address output) = (path[i], path[i + 1]);
            (address token0,) = UniswapV2Library.sortTokens(input, output);
            IUniswapV2Pair pair = IUniswapV2Pair(UniswapV2Library.pairFor(factory, input, output));
            uint amountInput;
            uint amountOutput;
            { // scope to avoid stack too deep errors
            (uint reserve0, uint reserve1,) = pair.getReserves();
            (uint reserveInput, uint reserveOutput) = input == token0 ? (reserve0, reserve1) : (reserve1, reserve0);
            amountInput = IERC20(input).balanceOf(address(pair)).sub(reserveInput);
            amountOutput = UniswapV2Library.getAmountOut(amountInput, reserveInput, reserveOutput);
```

Because of transfer fees, we cannot rely on `getAmountsOut` to tell us how much will be received after each transfer, which is what the regular `_swap` path assumes. Instead, the router must perform the transfer first and then inspect how many tokens were actually received.

Note: in theory, this function could be used instead of `_swap` for all swaps. In some cases, however, it would consume more gas, especially if the transaction reverts only at the end because the required minimum output cannot be met. Fee-on-transfer tokens are uncommon, so the router supports them without forcing every swap to assume that at least one token charges transfer fees.

```solidity
            }
            (uint amount0Out, uint amount1Out) = input == token0 ? (uint(0), amountOutput) : (amountOutput, uint(0));
            address to = i < path.length - 2 ? UniswapV2Library.pairFor(factory, output, path[i + 2]) : _to;
            pair.swap(amount0Out, amount1Out, to, new bytes(0));
        }
    }


    function swapExactTokensForTokensSupportingFeeOnTransferTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external virtual override ensure(deadline) {
        TransferHelper.safeTransferFrom(
            path[0], msg.sender, UniswapV2Library.pairFor(factory, path[0], path[1]), amountIn
        );
        uint balanceBefore = IERC20(path[path.length - 1]).balanceOf(to);
        _swapSupportingFeeOnTransferTokens(path, to);
        require(
            IERC20(path[path.length - 1]).balanceOf(to).sub(balanceBefore) >= amountOutMin,
            'UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT'
        );
    }


    function swapExactETHForTokensSupportingFeeOnTransferTokens(
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    )
        external
        virtual
        override
        payable
        ensure(deadline)
    {
        require(path[0] == WETH, 'UniswapV2Router: INVALID_PATH');
        uint amountIn = msg.value;
        IWETH(WETH).deposit{value: amountIn}();
        assert(IWETH(WETH).transfer(UniswapV2Library.pairFor(factory, path[0], path[1]), amountIn));
        uint balanceBefore = IERC20(path[path.length - 1]).balanceOf(to);
        _swapSupportingFeeOnTransferTokens(path, to);
        require(
            IERC20(path[path.length - 1]).balanceOf(to).sub(balanceBefore) >= amountOutMin,
            'UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT'
        );
    }


    function swapExactTokensForETHSupportingFeeOnTransferTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    )
        external
        virtual
        override
        ensure(deadline)
    {
        require(path[path.length - 1] == WETH, 'UniswapV2Router: INVALID_PATH');
        TransferHelper.safeTransferFrom(
            path[0], msg.sender, UniswapV2Library.pairFor(factory, path[0], path[1]), amountIn
        );
        _swapSupportingFeeOnTransferTokens(path, address(this));
        uint amountOut = IERC20(WETH).balanceOf(address(this));
        require(amountOut >= amountOutMin, 'UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT');
        IWETH(WETH).withdraw(amountOut);
        TransferHelper.safeTransferETH(to, amountOut);
    }
```

These variants mirror the regular swap functions, except that they call `_swapSupportingFeeOnTransferTokens`.

```solidity
    // **** LIBRARY FUNCTIONS ****
    function quote(uint amountA, uint reserveA, uint reserveB) public pure virtual override returns (uint amountB) {
        return UniswapV2Library.quote(amountA, reserveA, reserveB);
    }

    function getAmountOut(uint amountIn, uint reserveIn, uint reserveOut)
        public
        pure
        virtual
        override
        returns (uint amountOut)
    {
        return UniswapV2Library.getAmountOut(amountIn, reserveIn, reserveOut);
    }

    function getAmountIn(uint amountOut, uint reserveIn, uint reserveOut)
        public
        pure
        virtual
        override
        returns (uint amountIn)
    {
        return UniswapV2Library.getAmountIn(amountOut, reserveIn, reserveOut);
    }

    function getAmountsOut(uint amountIn, address[] memory path)
        public
        view
        virtual
        override
        returns (uint[] memory amounts)
    {
        return UniswapV2Library.getAmountsOut(factory, amountIn, path);
    }

    function getAmountsIn(uint amountOut, address[] memory path)
        public
        view
        virtual
        override
        returns (uint[] memory amounts)
    {
        return UniswapV2Library.getAmountsIn(factory, amountOut, path);
    }
}
```

These functions are simple proxies to the corresponding [UniswapV2Library functions](README.md#uniswapV2library).

#### UniswapV2Migrator.sol <a href="#uniswapv2migrator" id="uniswapv2migrator"></a>

This contract was used to migrate liquidity from the older v1 system to v2. The migration period has long passed, so it is no longer relevant for normal use.
