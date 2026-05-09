# Uniswap V2 Pair

### Core and Periphery Contracts <a href="#contract-types" id="contract-types"></a>

Uniswap v2 can be divided into two parts: the core and the periphery. This division keeps the core contracts, which hold assets and therefore _must_ be secure, smaller and easier to audit. Other features needed by traders can be provided through periphery contracts.

### Data and Control Flow <a href="#flows" id="flows"></a>

The following data and control flows appear when executing the three main Uniswap operations:

1. Swapping different tokens.
2. Adding funds to a market to provide liquidity and receiving liquidity-pool ERC-20 tokens as the reward-bearing claim.
3. Burning liquidity-pool ERC-20 tokens and receiving back the ERC-20 tokens that the exchange allows traders to swap.

#### Swap <a href="#swap-flow" id="swap-flow"></a>

This is the flow most commonly used by traders.

**Caller**

1. Provides an allowance to the periphery account for the tokens to be swapped.
2. Calls one of the swap functions in the periphery contract. The periphery contract usually has several swap functions. Which one is called depends on whether Ether is involved, whether the trader specifies the token amount to input, or specifies the token amount to receive, and so on. Each swap function accepts a `path`, meaning the sequence of swaps to execute.

**In the periphery contract (UniswapV2Router02.sol)**

1. Determines the token amounts required for each swap along the swap path.
2. Iterates along the path. For each swap on the path, it first sends the input token and then calls the pair exchange's `swap` function. In most cases, the destination address for the output token is the next pair in the path. In the final swap, that address is the address supplied by the trader.

**In the core contract (UniswapV2Pair.sol)**

1. Optimistically sends the output tokens to the destination address first; if `data` is not empty, it also triggers the flash-swap callback.
2. Reads the Pair's current balances and infers how many tokens were actually input in this swap from the difference between the balances and the old reserves.
3. Checks the constant-product invariant using balances after fee deduction, ensuring that the core contract has not been tricked.
4. Calls `_update` to update the reserves.

**Back in the periphery contract (UniswapV2Router02.sol)**

1. Performs any necessary cleanup, such as burning wrapped Ether tokens to return Ether to the trader.

#### Add Liquidity <a href="#add-liquidity-flow" id="add-liquidity-flow"></a>

**Caller**

1. Gives the periphery account an allowance for the funds to be added to the liquidity pool.
2. Calls one of the periphery contract's `addLiquidity` functions.

**In the periphery contract (UniswapV2Router02.sol)**

1. Creates a new pair if necessary.
2. If an existing pair is present, calculates the token amounts to add. The two tokens should have equal value, so the ratio of newly added tokens should match the ratio of existing tokens.
3. Checks whether the amounts are acceptable. The caller can specify minimum amounts below which they refuse to add liquidity.
4. Calls the core contract.

**In the core contract (UniswapV2Pair.sol)**

1. Mints liquidity-pool tokens and sends them to the caller.
2. Calls `_update` to update the reserves.

#### Remove Liquidity <a href="#remove-liquidity-flow" id="remove-liquidity-flow"></a>

**Caller**

1. Gives the periphery account an allowance for the liquidity-pool tokens that must be burned in exchange for the underlying tokens.
2. Calls one of the periphery contract's `removeLiquidity` functions.

**In the periphery contract (UniswapV2Router02.sol)**

1. Sends the liquidity-pool tokens to the pair.

**In the core contract (UniswapV2Pair.sol)**

1. Sends the redeemed tokens to the target address according to the proportion of liquidity tokens burned. For example, if the pool contains 1,000 A tokens, 500 B tokens, and 90 liquidity-pool tokens, and we are asked to burn 9 liquidity-pool tokens, then 10% of the liquidity-pool tokens are burned and the user receives 100 A tokens and 50 B tokens.
2. Burns the liquidity-pool tokens.
3. Calls `_update` to update the reserves.

### Core Contracts <a href="#core-contracts" id="core-contracts"></a>

These are the security-critical contracts that hold liquidity.

#### UniswapV2Pair.sol <a href="#uniswapv2pair" id="uniswapv2pair"></a>

[This contract](https://github.com/Uniswap/uniswap-v2-core/blob/master/contracts/UniswapV2Pair.sol) implements the actual pool used for token swaps. This is Uniswap's core functionality.

```solidity
pragma solidity =0.5.16;

import './interfaces/IUniswapV2Pair.sol';
import './UniswapV2ERC20.sol';
import './libraries/Math.sol';
import './libraries/UQ112x112.sol';
import './interfaces/IERC20.sol';
import './interfaces/IUniswapV2Factory.sol';
import './interfaces/IUniswapV2Callee.sol';
```

These are the interfaces the contract needs to know about, either because the contract implements them (`IUniswapV2Pair` and `UniswapV2ERC20`) or because it calls contracts that implement them.

```solidity
contract UniswapV2Pair is IUniswapV2Pair, UniswapV2ERC20 {
```

This contract inherits from `UniswapV2ERC20`, which provides ERC-20 token functionality for the liquidity-pool token.

```solidity
    using SafeMath  for uint;
```

The [SafeMath library](https://docs.openzeppelin.com/contracts/3.x/api/math) is used to avoid integer overflow and underflow. This is important; otherwise, a value that should be `-1` might become `2^256-1`.

```solidity
    using UQ112x112 for uint224;
```

Many calculations in the liquidity-pool contract require fractions. However, the Ethereum Virtual Machine does not natively support fractions. The solution used by Uniswap is to use a 224-bit value, with 112 bits for the integer part and 112 bits for the fractional part. Therefore, `1.0` is represented as `2^112`, `1.5` is represented as `2^112 + 2^111`, and so on.

This library is discussed in more detail later in the [documentation](README.md#FixedPoint).

**Variables**

```solidity
    uint public constant MINIMUM_LIQUIDITY = 10**3;
```

To avoid a zero denominator, a minimum number of liquidity-pool tokens always exists, but they are owned by the zero address. This number, **MINIMUM_LIQUIDITY**, is 1000.

```solidity
    bytes4 private constant SELECTOR = bytes4(keccak256(bytes('transfer(address,uint256)')));
```

This is the application binary interface selector for the ERC-20 transfer function. It is used to transfer ERC-20 tokens between token accounts.

```solidity
    address public factory;
```

This is the address of the factory contract that created the pool. Each pool is an exchange between two ERC-20 tokens, and the factory is the central registry connecting all these token pools.

```solidity
    address public token0;
    address public token1;
```

These two addresses are the contract addresses of the two ERC-20 token types that the liquidity pool can swap.

```solidity
    uint112 private reserve0;           // uses single storage slot, accessible via getReserves
    uint112 private reserve1;           // uses single storage slot, accessible via getReserves
```

Each token type has its own reserve. We assume the two reserves represent equal total value, so each unit of token0 is worth `reserve1/reserve0` units of token1.

```solidity
    uint32  private blockTimestampLast; // uses single storage slot, accessible via getReserves
```

This is the timestamp of the last block in which an exchange occurred. It is used to track the exchange rate over time.

One of the largest gas costs in Ethereum contracts is storage, because storage persists from one contract call to the next. Each storage slot is 256 bits. Therefore, `reserve0`, `reserve1`, and `blockTimestampLast` are laid out so a single storage slot can contain all three variables: 112 + 112 + 32 = 256.

```solidity
    uint public price0CumulativeLast;
    uint public price1CumulativeLast;
```

These variables store the cumulative price of each token in terms of the other token. They can be used to calculate the average exchange rate over a period of time.

```solidity
    uint public kLast; // reserve0 * reserve1, as of immediately after the most recent liquidity event
```

In a pair, the exchange rate between token0 and token1 is determined by keeping the product of the two reserves constant. That product is `kLast`. It changes when liquidity providers deposit or withdraw tokens, and it also increases slightly because swaps pay a 0.3% market fee.

Here is an example. For simplicity, the numbers in the table keep only three decimal places, and the 0.3% trading fee is ignored, so the numbers are not exact.

| Event | reserve0 | reserve1 | reserve0 * reserve1 | Average exchange rate (token1 / token0) |
| ---- | --------: | --------: | -------------------: | ---------------------- |
| Initial setup | 1,000.000 | 1,000.000 | 1,000,000 | |
| Trader A swaps 50 token0 for 47.619 token1 | 1,050.000 | 952.381 | 1,000,000 | 0.952 |
| Trader B swaps 10 token0 for 8.984 token1 | 1,060.000 | 943.396 | 1,000,000 | 0.898 |
| Trader C swaps 40 token0 for 34.305 token1 | 1,100.000 | 909.090 | 1,000,000 | 0.858 |
| Trader D swaps 100 token1 for 109.01 token0 | 990.990 | 1,009.090 | 1,000,000 | 0.917 |
| Trader E swaps 10 token0 for 10.079 token1 | 1,000.990 | 999.010 | 1,000,000 | 1.008 |

As traders provide more token0, the relative value of token1 increases, and vice versa. This is driven by supply and demand.

**Lock**

```solidity
    uint private unlocked = 1;
```

There is a class of security issues based on [reentrancy attacks](https://medium.com/coinmonks/ethernaut-lvl-10-re-entrancy-walkthrough-how-to-abuse-execution-ordering-and-reproduce-the-dao-7ec88b912c14). Uniswap needs to transfer different ERC-20 token amounts, which means the called ERC-20 contract might try to attack the Uniswap market that called it. By using the `unlocked` variable, we can prevent a function from being called while it is already running in the same transaction.

```solidity
    modifier lock() {
```

This function is a [modifier](https://docs.soliditylang.org/en/v0.8.3/contracts.html#function-modifiers), which changes normal function behavior in a specific way.

```solidity
        require(unlocked == 1, 'UniswapV2: LOCKED');
        unlocked = 0;
```

If `unlocked` is 1, set it to 0. If it is already 0, revert the call and fail.

```solidity
        _;
```

Inside a modifier, `_;` represents the original function call with all its parameters. Here it means the function can only be called when `unlocked` is 1, and while the function is running, `unlocked` is 0.

```solidity
        unlocked = 1;
    }
```

After the main function returns, release the lock.

**Other Functions**

```solidity
    function getReserves() public view returns (uint112 _reserve0, uint112 _reserve1, uint32 _blockTimestampLast) {
        _reserve0 = reserve0;
        _reserve1 = reserve1;
        _blockTimestampLast = blockTimestampLast;
    }
```

This function returns the current exchange state to the caller. Note that Solidity functions [can return multiple values](https://docs.soliditylang.org/en/v0.8.3/contracts.html#returning-multiple-values).

```solidity
    function _safeTransfer(address token, address to, uint value) private {
        (bool success, bytes memory data) = token.call(abi.encodeWithSelector(SELECTOR, to, value));
```

This internal function transfers a certain amount of ERC-20 tokens from the exchange to another account. `SELECTOR` specifies that the function being called is `transfer(address,uint)`, as defined above.

To avoid importing an interface for token functions, the contract uses one of the [ABI functions](https://docs.soliditylang.org/en/v0.8.3/units-and-global-variables.html#abi-encoding-and-decoding-functions) to manually create the call.

```solidity
        require(success && (data.length == 0 || abi.decode(data, (bool))), 'UniswapV2: TRANSFER_FAILED');
    }
```

An ERC-20 transfer call can fail in two ways:

1. It reverts. If the external contract call reverts, the boolean return value is `false`.
2. It completes normally but reports failure. In this case, the returned buffer has non-zero length, and when decoded as a boolean, its value is `false`.

If either case occurs, the transfer call is reverted.

**Events**

```solidity
    event Mint(address indexed sender, uint amount0, uint amount1);
    event Burn(address indexed sender, uint amount0, uint amount1, address indexed to);
```

These two events are emitted when a liquidity provider deposits liquidity (`Mint`) or withdraws liquidity (`Burn`). In both cases, the amounts of token0 and token1 deposited or withdrawn are part of the event, as is the account address that called the contract (`sender`). When withdrawing liquidity, the event also includes the target address (`to`) that receives the tokens. This address may differ from the account address that sent the contract call.

```solidity
    event Swap(
        address indexed sender,
        uint amount0In,
        uint amount1In,
        uint amount0Out,
        uint amount1Out,
        address indexed to
    );
```

This event is emitted when a trader swaps one token for another. Again, the token sender and the destination account that receives the swapped tokens may be different. Each token can either be sent to the exchange or received from the exchange.

```solidity
    event Sync(uint112 reserve0, uint112 reserve1);
```

Finally, `Sync` is emitted every time tokens are deposited or withdrawn for any reason, providing the latest reserve information and therefore the exchange rate.

**Setup Functions**

These functions should be called when a new pair is created.

```solidity
    constructor() public {
        factory = msg.sender;
    }
```

The constructor ensures that the pair can track the address of the factory contract that created it. The `initialize` function and the factory contract fee setting, if any, need this information.

```solidity
    // called once by the factory at time of deployment
    function initialize(address _token0, address _token1) external {
        require(msg.sender == factory, 'UniswapV2: FORBIDDEN'); // sufficient check
        token0 = _token0;
        token1 = _token1;
    }
```

This function allows the factory, and only the factory, to specify the two ERC-20 tokens traded by the pair.

**Internal Update Functions**

**_update**

```solidity
    // update reserves and, on the first call per block, price accumulators
    function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
```

This function is called every time tokens are deposited or withdrawn.

```solidity
        require(balance0 <= uint112(-1) && balance1 <= uint112(-1), 'UniswapV2: OVERFLOW');
```

If `balance0` or `balance1`, both `uint256`, is greater than `uint112(-1)` (= 2^112 - 1), converting it to `uint112` would overflow. The call is rejected to prevent overflow. Typical tokens can be subdivided into 10^18 units, which means the exchange limit is about 5.1 * 10^15 units of each token. So far, this has not been a practical issue.

```solidity
        uint32 blockTimestamp = uint32(block.timestamp % 2**32);
        uint32 timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
        if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
```

If `timeElapsed` is not zero, this transaction is the first exchange transaction in this block. In that case, the cumulative price values need to be updated.

```solidity
            // * never overflows, and + overflow is desired
            price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
            price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
        }
```

Each cumulative price value is updated by adding the latest price, meaning the reserve amount of the other token divided by the reserve amount of this token, multiplied by elapsed time in seconds. To obtain the average exchange price, read two cumulative price values and divide their difference by the time difference between the two observations. For example, suppose the following event sequence:

| Event | reserve0 | reserve1 | Timestamp | Marginal exchange rate (reserve1 / reserve0) | price0CumulativeLast |
| ---- | --------: | --------: | ----- | -------------------------: | -------------------------: |
| Initial setup | 1,000.000 | 1,000.000 | 5,000 | 1.000 | 0 |
| Trader A deposits 50 token0 and receives 47.619 token1 | 1,050.000 | 952.381 | 5,020 | 0.907 | 20 |
| Trader B deposits 10 token0 and receives 8.984 token1 | 1,060.000 | 943.396 | 5,030 | 0.89 | 20+10*0.907 = 29.07 |
| Trader C deposits 40 token0 and receives 34.305 token1 | 1,100.000 | 909.090 | 5,100 | 0.826 | 29.07+70*0.890 = 91.37 |
| Trader D deposits 100 token0 and receives 109.01 token1 | 990.990 | 1,009.090 | 5,110 | 1.018 | 91.37+10*0.826 = 99.63 |
| Trader E deposits 10 token0 and receives 10.079 token1 | 1,000.990 | 999.010 | 5,150 | 0.998 | 99.63+40*1.1018 = 143.702 |

For example, suppose we want to calculate the average price of **token0** between timestamp 5,030 and 5,150. The difference in `price0Cumulative` is 143.702 - 29.07 = 114.632. This covers two minutes, or 120 seconds. Therefore, the average price is 114.632 / 120 = 0.955.

This price calculation is why we need to know the old reserve sizes.

```solidity
        reserve0 = uint112(balance0);
        reserve1 = uint112(balance1);
        blockTimestampLast = blockTimestamp;
        emit Sync(reserve0, reserve1);
    }
```

Finally, the global variables are updated and a `Sync` event is emitted.

**_mintFee**

```solidity
    // if fee is on, mint liquidity equivalent to 1/6th of the growth in sqrt(k)
    function _mintFee(uint112 _reserve0, uint112 _reserve1) private returns (bool feeOn) {
```

The Uniswap 2.0 contracts specify that traders pay a 0.30% fee to use the exchange market. Most of this fee, 0.25% of the trade, is paid to liquidity providers. The remaining 0.05% may be paid either to liquidity providers or to the account specified by the factory contract as the protocol fee, which can be used to fund Uniswap development.

To reduce the number of computations and therefore gas cost, this fee is calculated only when liquidity is added or removed, not on every swap.

```solidity
        address feeTo = IUniswapV2Factory(factory).feeTo();
        feeOn = feeTo != address(0);
```

Read the factory's fee recipient address. If the returned value is zero, there is no protocol fee and no fee needs to be calculated.

```solidity
        uint _kLast = kLast; // gas savings
```

The `kLast` state variable is in storage, so it has a value across contract calls. Function memory is cleared after each function call, but because storage access is much more expensive than memory access, this local memory variable is used to represent the storage value and reduce gas cost.

```solidity
        if (feeOn) {
            if (_kLast != 0) {
```

Liquidity providers earn fees simply because they provide liquidity tokens. The protocol fee, however, requires minting new liquidity tokens and giving them to the `feeTo` account.

```solidity
                uint rootK = Math.sqrt(uint(_reserve0).mul(_reserve1));
                uint rootKLast = Math.sqrt(_kLast);
                if (rootK > rootKLast) {
```

If there has been new liquidity growth, a protocol fee may need to be charged. The square-root function is discussed [later in this document](README.md#Math).

```solidity
                    uint numerator = totalSupply.mul(rootK.sub(rootKLast));
                    uint denominator = rootK.mul(5).add(rootKLast);
                    uint liquidity = numerator / denominator;
```

This complicated fee calculation is explained on page 5 of the [whitepaper](https://uniswap.org/whitepaper.pdf). During the interval in which `kLast` is calculated, liquidity has not changed, because every calculation occurs when liquidity actually changes. Therefore, the change in `reserve0 * reserve1` must come from trading fees; without trading fees, `reserve0 * reserve1` would remain constant.

```solidity
                    if (liquidity > 0) _mint(feeTo, liquidity);
                }
            }
```

Use the `UniswapV2ERC20._mint` function to create more liquidity-pool tokens and send them to the `feeTo` address.

```solidity
        } else if (_kLast != 0) {
            kLast = 0;
        }
    }
```

If no fee is required, set `kLast` to 0 if it is not already 0. When this contract was written, there was a [gas refund feature](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-3298.md) that encouraged contracts to release storage they no longer needed, reducing the total state stored on Ethereum. This code claimed the refund when possible.

**Externally Accessible Functions**

Although these functions _can_ be called by any transaction or contract, they are designed to be called by external contracts. If called directly, you cannot trick the pair, and you may lose value due to mistakes.

**Mint**

```solidity
    // this low-level function should be called from a contract which performs important safety checks
    function mint(address to) external lock returns (uint liquidity) {
```

This function is called when a liquidity provider adds liquidity to the pool. It mints additional liquidity-pool tokens as the reward. The function is called after liquidity is added in the [periphery contract](README.md#UniswapV2Router02), ensuring both actions happen in the same transaction, so no one else can submit a transaction claiming the new liquidity from the legitimate owner.

```solidity
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
```

This is how Solidity reads multiple return values. Here the final returned value, the block timestamp, is ignored because it is not needed.

```solidity
        uint balance0 = IERC20(token0).balanceOf(address(this));
        uint balance1 = IERC20(token1).balanceOf(address(this));
        uint amount0 = balance0.sub(_reserve0);
        uint amount1 = balance1.sub(_reserve1);
```

Get the current balances and see how much of each token type was added.

```solidity
        bool feeOn = _mintFee(_reserve0, _reserve1);
```

If a protocol fee exists, calculate the fee to be collected and mint liquidity-pool tokens accordingly. Because the inputs to `_mintFee` are the old reserve values, the fee calculation is based only on pool growth caused by fees.

```solidity
        uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
        if (_totalSupply == 0) {
            liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
           _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
```

If this is the first deposit, `MINIMUM_LIQUIDITY` tokens are created and sent to address 0 for permanent locking. These tokens cannot be retrieved, meaning the liquidity pool is never empty, which avoids division-by-zero errors in later calculations. `MINIMUM_LIQUIDITY` is 1000. Given that most ERC-20 tokens are subdivided into 10^-18 units of one token, while Ether is divided into wei, this is only 10^-15 of a token and is not expensive.

On the first deposit, the relative value of the two tokens is not known, so the contract assumes the two token amounts have equal value and simply takes the square root of their product.

We can rely on this because it is in the depositor's interest to provide equal value and avoid arbitrage. Suppose the two tokens have the same value, but the depositor deposits four times as much **Token1** as **Token0**. Through the pair, traders can treat **Token0** as more valuable.

| Event | reserve0 | reserve1 | reserve0 * reserve1 | Pool value (reserve0 + reserve1) |
| ---- | -------: | -------: | -------------------: | --------------------------: |
| Initial setup | 8 | 32 | 256 | 40 |
| Trader deposits 8 **Token0** and receives 16 **Token1** | 16 | 16 | 256 | 32 |

As shown, the trader receives 8 extra tokens. This comes from a decrease in pool value and harms the depositor who owns the pool.

```solidity
        } else {
            liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);
```

For every later deposit, the exchange rate between the two assets is already known. We expect liquidity providers to provide equal value in the two tokens. If they do not, they are penalized by receiving liquidity-pool tokens based on the lower-value token amount they provided.

For both the initial deposit and later deposits, the amount of liquidity-pool tokens is equal to the square root of `reserve0*reserve1`, and the value of each liquidity-pool token remains constant, unless the deposit contains unequal-value token types, in which case a "penalty" is allocated. Here is another example where the two tokens have equal value, with three normal deposits and one malicious deposit, meaning only one token type is deposited and therefore no liquidity-pool tokens are created.

| Event | reserve0 | reserve1 | reserve0 * reserve1 | Pool value (reserve0 + reserve1) | Liquidity-pool tokens minted by deposit | Total liquidity-pool tokens | Value per liquidity-pool token |
| ---- | -------: | -------: | -------------------: | --------------------------: | ------------: | ------: | --------: |
| Initial setup | 8.000 | 8.000 | 64 | 16.000 | 8 | 8 | 2.000 |
| Deposit 4 of each token | 12.000 | 12.000 | 144 | 24.000 | 4 | 12 | 2.000 |
| Deposit 2 of each token | 14.000 | 14.000 | 196 | 28.000 | 2 | 14 | 2.000 |
| Unequal-value deposit | 18.000 | 14.000 | 252 | 32.000 | 0 | 14 | ~2.286 |
| After arbitrage | ~15.874 | ~15.874 | 252 | ~31.748 | 0 | 14 | ~2.267 |

```solidity
        }
        require(liquidity > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_MINTED');
        _mint(to, liquidity);
```

Use the `UniswapV2ERC20._mint` function to create more liquidity-pool tokens and send them to the correct account address.

```solidity
        _update(balance0, balance1, _reserve0, _reserve1);
        if (feeOn) kLast = uint(reserve0).mul(reserve1); // reserve0 and reserve1 are up-to-date
        emit Mint(msg.sender, amount0, amount1);
    }
```

Update the corresponding state variables, `reserve0`, `reserve1`, and, if necessary, `kLast`, and emit the corresponding event.

**Burn**

```solidity
    // this low-level function should be called from a contract which performs important safety checks
    function burn(address to) external lock returns (uint amount0, uint amount1) {
```

This function is called when liquidity is withdrawn and the corresponding liquidity-pool tokens need to be burned. It should also be called [from a periphery account](README.md#UniswapV2Router02).

```solidity
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
        address _token0 = token0;                                // gas savings
        address _token1 = token1;                                // gas savings
        uint balance0 = IERC20(_token0).balanceOf(address(this));
        uint balance1 = IERC20(_token1).balanceOf(address(this));
        uint liquidity = balanceOf[address(this)];
```

Before calling this function, the periphery contract first transfers the liquidity to be burned into this contract. This lets the pair know how much liquidity should be burned and ensures that it can be burned.

```solidity
        bool feeOn = _mintFee(_reserve0, _reserve1);
        uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
        amount0 = liquidity.mul(balance0) / _totalSupply; // using balances ensures pro-rata distribution
        amount1 = liquidity.mul(balance1) / _totalSupply; // using balances ensures pro-rata distribution
        require(amount0 > 0 && amount1 > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_BURNED');
```

The liquidity provider receives equal-value amounts of both tokens. This does not change the exchange rate.

```solidity
        _burn(address(this), liquidity);
        _safeTransfer(_token0, to, amount0);
        _safeTransfer(_token1, to, amount1);
        balance0 = IERC20(_token0).balanceOf(address(this));
        balance1 = IERC20(_token1).balanceOf(address(this));

        _update(balance0, balance1, _reserve0, _reserve1);
        if (feeOn) kLast = uint(reserve0).mul(reserve1); // reserve0 and reserve1 are up-to-date
        emit Burn(msg.sender, amount0, amount1, to);
    }
```

The rest of the `burn` function mirrors the `mint` function described above.

**Swap**

```solidity
    // this low-level function should be called from a contract which performs important safety checks
    function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external lock {
```

This function should also be called from the [periphery contract](README.md#UniswapV2Router02).

```solidity
        require(amount0Out > 0 || amount1Out > 0, 'UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT');
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
        require(amount0Out < _reserve0 && amount1Out < _reserve1, 'UniswapV2: INSUFFICIENT_LIQUIDITY');

        uint balance0;
        uint balance1;
        { // scope for _token{0,1}, avoids stack too deep errors
```

Local variables can be stored in memory, or, if there are not too many variables, directly on the stack. If the number of variables can be limited, using the stack reduces gas consumption. For details, see the [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf), formerly the Ethereum specification, p. 26, "Equation 298".

```solidity
            address _token0 = token0;
            address _token1 = token1;
            require(to != _token0 && to != _token1, 'UniswapV2: INVALID_TO');
            if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out); // optimistically transfer tokens
            if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out); // optimistically transfer tokens
```

These transfers should succeed because all required conditions have already been checked before the transfer. This is safe in Ethereum because if later conditions are not satisfied, the transaction can revert and undo the changes.

```solidity
            if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);
```

If requested, notify the receiver that a swap is being performed.

```solidity
            balance0 = IERC20(_token0).balanceOf(address(this));
            balance1 = IERC20(_token1).balanceOf(address(this));
        }
```

Read the current balances. The periphery contract must send the token being swapped to the pair before calling the swap function. This allows the contract to verify that it has not been tricked. This check _must_ be performed by the core contract, because the function may be called by entities other than the intended periphery contract.

```solidity
        uint amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
        uint amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
        require(amount0In > 0 || amount1In > 0, 'UniswapV2: INSUFFICIENT_INPUT_AMOUNT');
        { // scope for reserve{0,1}Adjusted, avoids stack too deep errors
            uint balance0Adjusted = balance0.mul(1000).sub(amount0In.mul(3));
            uint balance1Adjusted = balance1.mul(1000).sub(amount1In.mul(3));
            require(balance0Adjusted.mul(balance1Adjusted) >= uint(_reserve0).mul(_reserve1).mul(1000**2), 'UniswapV2: K');
```

This is a sanity check that ensures the swap does not make the pair lose tokens. Under no circumstances should a swap reduce `reserve0*reserve1`. This is also how the contract ensures the 0.3% swap fee is paid: before checking the K invariant, both balances are multiplied by 1000 and three times the input amount is subtracted. This deducts 0.3% from the input balance, because 3/1000 = 0.003 = 0.3%, before comparing the adjusted K value with the current reserve K value.

```solidity
        }

        _update(balance0, balance1, _reserve0, _reserve1);
        emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
    }
```

Update `reserve0` and `reserve1`, and when necessary update the price cumulative values and timestamp, then emit the corresponding event.

**Sync or Skim**

The actual balances may become unsynchronized with the reserves that the pair believes it has. Tokens cannot be withdrawn without the contract's consent, but deposits are different. An account can transfer tokens into the exchange without calling `mint` or `swap`.

In this situation, there are two possible solutions:

* `sync`: update the reserves to the current balances.
* `skim`: withdraw the extra amount. Note that any account can call `skim`, because there is no way to know who deposited the tokens. That information is published in an event, but events cannot be read from the blockchain by contracts.

```solidity
    // force balances to match reserves
    function skim(address to) external lock {
        address _token0 = token0; // gas savings
        address _token1 = token1; // gas savings
        _safeTransfer(_token0, to, IERC20(_token0).balanceOf(address(this)).sub(reserve0));
        _safeTransfer(_token1, to, IERC20(_token1).balanceOf(address(this)).sub(reserve1));
    }



    // force reserves to match balances
    function sync() external lock {
        _update(IERC20(token0).balanceOf(address(this)), IERC20(token1).balanceOf(address(this)), reserve0, reserve1);
    }
}
```
