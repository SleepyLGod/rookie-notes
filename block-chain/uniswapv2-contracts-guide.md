# 😈 Uniswap-v2 合约概览

### 介绍 <a href="#introduction" id="introduction"></a>

[Uniswap v2](https://uniswap.org/whitepaper.pdf) 可以在任何两个 ERC-20 代币之间创建一个兑换市场。 在这篇文章中， 我们将了解实现此协议的合约的源代码，看看为什么要 这样写代码。

#### Uniswap 是做什么的？ <a href="#what-does-uniswap-do" id="what-does-uniswap-do"></a>

一般来说有两类用户：流动资金提供者和交易者。

_流动资金提供者\_为资金池提供两种可以兑换的代币（我们称之为 **Token0** 和 **Token1**）。 作为回报，他们会收到第三种代币，代表对资金池的 部分所有权，这个池叫做\_流动代币_。

\_交易者\_将一种代币发送到资金池，并从流动资金提供者的资金池中接收另一种代币（例如，发送 **Token0** 并获得 **Token1**）。 兑换汇率由 **Token0** 和 **Token1** 的相对数量决定。 此外，资金池将收取汇率的一小部分作为流动资金池的奖励。

当流动资金提供者想要收回他们的代币资产时，他们可以消耗资金池代币并收回他们的代币， 其中包括他们在兑换过程中奖励的份额。

[点击这里查看更完整的描述](https://uniswap.org/docs/v2/core-concepts/swaps/)。

#### 为什么选择 v2？ 而不是 v3？ <a href="#why-v2" id="why-v2"></a>

编写此教程时，[Uniswap v3](https://uniswap.org/whitepaper-v3.pdf) 已差不多准备就绪。 然而，此次升级 比原来的版本复杂得多。 比较容易的方法是先学习 v2，然后再学习 v3。

#### 核心合约与外围合约 <a href="#contract-types" id="contract-types"></a>

Uniswap v2 可以分为两个部分，一个为核心部分，另一个为外围部分。 这种分法可以使拥有资产因而\_必须\_确保安全的核心合约更加简洁，且更易于审核。 而所有交易者需要的其它功能可以通过外围合约提供。

### 数据和控制流程 <a href="#flows" id="flows"></a>

执行 Uniswap 的三个主要操作时，会出现以下数据和控制流程：

1. 兑换不同代币
2. 将资金添加到市场中提供流动性，并获得兑换中奖励的流动池 ERC-20 代币
3. 消耗流动池 ERC-20 代币并收回交易所允许交易者兑换的 ERC-20 代币

#### 兑换 <a href="#swap-flow" id="swap-flow"></a>

这是交易者最常用的流程：

**调用者**

1. 向外围帐户提供兑换额度。
2. 调用外围合约中的一个兑换函数。外围合约通常会有多种兑换函数，调用哪一个取决于是否涉及以太币、 交易者是否需要指定存入的代币金额，或指定提取的代币数量等）。 每个兑换函数都接受一个 `path`，即要执行的一系列兑换。

**在外围合约 (UniswapV2Router02.sol) 中**

1. 确定兑换路径中，每次兑换所需交易的代币数额。
2. 沿路径迭代。 对于路径上的每次兑换，首先发送输入代币，然后调用交易所的 `swap` 函数。 在大多数情况下，代币输出的目的地址是路径中下一个配对交易。 在最终的兑换中，该地址是 交易者提供的地址。

**在核心合约 (UniswapV2Pair.sol) 中**

1. 验证核心合约没有被欺骗，可在兑换后保持足够的流动资金。
2. 检查除了现有的储备金额外，还有多少额外的代币。 此数额是我们收到的要用于兑换的输入代币数量。
3. 将输出代币发送到目的地址。
4. 调用 `_update` 来更新储备金额

**回到外围合约 (UniswapV2Router02.sol)**

1. 执行所需的必要清理工作（例如，消耗包装以太币代币以返回以太币给交易者）

#### 增加流动资金 <a href="#add-liquidity-flow" id="add-liquidity-flow"></a>

**调用者**

1. 向外围账户提交准备加入流动资金池的资金额度。
2. 调用外围合约的一个 addLiquidity 函数。

**在外围合约 (UniswapV2Router02.sol) 中**

1. 必要时创建一个新的配对交易
2. 如果存在现有配对交易，请计算要增加的代币数量。 两个代币应该有相同值，所以新代币与现有代币的比率是相同的。
3. 检查金额是否可接受（调用者可以指定一个最低金额，低于此金额他们不能增加流动资金）
4. 调用核心合约。

**在核心合约 (UniswapV2Pair.sol) 中**

1. 生成流动池代币并将其发送给调用者
2. 调用 `_update` 来更新储备金额

#### 撤回流动资金 <a href="#remove-liquidity-flow" id="remove-liquidity-flow"></a>

**调用者**

1. 向外围帐户提供一个流动池代币的额度，作为兑换底层代币所需的消耗。
2. 调用外围合约的一个 removeLiquidity 函数。

**在外围合约 (UniswapV2Router02.sol) 中**

1. 将流动池代币发送到该配对交易

**在核心合约 (UniswapV2Pair.sol) 中**

1. 按照消耗代币的比例发送兑换后的代币到目标地址。 例如，如果 流动池里有 1000 个 A 代币，500 个 B 代币和 90 个流动池代币，而我们被要求消耗 9 个 流动池代币，那么，我们将消耗 10% 的流动池代币，然后将返还用户 100 个 A 代币和 50 个 B 代币。
2. 消耗流动池代币
3. 调用`_update`来更新储备金额

### 核心合约 <a href="#core-contracts" id="core-contracts"></a>

这些是持有流动资金的安全合约。

#### UniswapV2Pair.sol <a href="#uniswapv2pair" id="uniswapv2pair"></a>

[本合约](https://github.com/Uniswap/uniswap-v2-core/blob/master/contracts/UniswapV2Pair.sol) 实现了 用于兑换代币的实际资金池。 这是 Uniswap 的核心功能。

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

这些都是合约需要知道的接口，因为合约实现了它们 （`IUniswapV2Pair` 和 `UniswapV2ERC20`），或因为合约调用了实现它们的合约。

```solidity
contract UniswapV2Pair is IUniswapV2Pair, UniswapV2ERC20 {
```

此合约继承自 `UniswapV2ERC20`，为流动池代币提供 ERC-20 代币功能。

```solidity
    using SafeMath  for uint;
```

[SafeMath 库](https://docs.openzeppelin.com/contracts/2.x/api/math)用于避免整数上溢和 下溢。 这很重要，否则最终可能会出现这样的情况：本该是 `-1` 的值， 结果却成了 `2^256-1`。

```solidity
    using UQ112x112 for uint224;
```

流动池合约中的许多计算都需要分数。 但是，以太坊虚拟机本身不支持分数。 Uniswap 找到的解决方案是使用 224 位数值，整数值为 112 位，分数部分 为 112 位。 因此，`1.0` 用 `2^112` 表示，`1.5` 用 `2^112 + 2^111` 表示，以此类推。

关于这个函数库的更详细内容在[文档的稍后部分](uniswapv2-contracts-guide.md#FixedPoint)。

**变量**

```solidity
    uint public constant MINIMUM_LIQUIDITY = 10**3;
```

为了避免分母为零的情况，最低数量的流动池代币总是存在的 （但为账户零所拥有）。 该数字，即 **MINIMUM\_LIQUIDITY**，为 1000。

```solidity
    bytes4 private constant SELECTOR = bytes4(keccak256(bytes('transfer(address,uint256)')));
```

这是 ERC-20 传输函数的应用程序二进制接口选择程序。 它用于在两个代币账户中转移 ERC-20 代币。

```solidity
    address public factory;
```

这就是由工厂合约创造的资金池地址。 每个资金池都是两个 ERC-20 代币之间的交换， 工厂是连接所有这些代币资金池的中心点。

```solidity
    address public token0;
    address public token1;
```

这两个地址是流动池可以兑换的 两类 ERC-20 代币的合约地址。

```solidity
    uint112 private reserve0;           // uses single storage slot, accessible via getReserves
    uint112 private reserve1;           // uses single storage slot, accessible via getReserves
```

每个代币类型都有储备的资源库。 我们假定两者代表相同数量的值， 因此每个 token0 的价值都等同于 reserve1/reserve0 token1。

```solidity
    uint32  private blockTimestampLast; // uses single storage slot, accessible via getReserves
```

发生兑换的最后一个区块的时间戳，用来追踪一段时间内的汇率。

以太坊合约中燃料消耗量最大的一项是存储，这种燃料消耗从一次合约调用持续到 下一次调用。 每个存储单元长度为 256 位。 因此，reserve0、reserve1 和 blockTimestampLast 三个变量的分配方式让 单个存储值可以包含全部这三个变量 (112+112+32=256)。

```solidity
    uint public price0CumulativeLast;
    uint public price1CumulativeLast;
```

这些变量存放每种代币的累计成本（每种代币在另一种代币的基础上计算）。 可以用来计算 一段时间内的平均汇率。

```solidity
    uint public kLast; // reserve0 * reserve1, as of immediately after the most recent liquidity event
```

在配对交易中，决定 token0 和 token1 之间汇率的方式是在交易中 保留两个储备常量的乘数。 即 `kLast` 这个值。 当流动资金提供者存入或提取代币时，它就会发生变化，由于兑换市场的费用为 0.3%，它会略有增加。

下面是一个示例。 请注意，为了简单起见，表格中的数字仅保留了小数点后三位，我们忽略了 0.3% 交易费，因此数字并不准确。

| 事件                                      |  reserve0 |  reserve1 | reserve0 \* reserve1 | 平均汇率 (token1 / token0) |
| --------------------------------------- | --------: | --------: | -------------------: | ---------------------- |
| 初始设置                                    | 1,000.000 | 1,000.000 |            1,000,000 |                        |
| 交易者 A 用 50 个 token0 兑换 47.619 个 token1  | 1,050.000 |   952.381 |            1,000,000 | 0.952                  |
| 交易者 B 用 10 个 token0 兑换 8.984 个 token1   | 1,060.000 |   943.396 |            1,000,000 | 0.898                  |
| 交易者 C 用 40 个 token0 兑换 34.305 个 token1  | 1,100.000 |   909.090 |            1,000,000 | 0.858                  |
| 交易者 D 用 100 个 token1 兑换 109.01 个 token0 |   990.990 | 1,009.090 |            1,000,000 | 0.917                  |
| 交易者 E 用 10 个 token0 兑换 10.079 个 token1  | 1,000.990 |   999.010 |            1,000,000 | 1.008                  |

由于交易者提供了更多 token0，token1 的相对价值增加了，反之亦然，这取决于供求。

**锁定**

```solidity
    uint private unlocked = 1;
```

有一类基于 [重入攻击](https://medium.com/coinmonks/ethernaut-lvl-10-re-entrancy-walkthrough-how-to-abuse-execution-ordering-and-reproduce-the-dao-7ec88b912c14)的安全问题。 Uniswap 需要转让不同数值的 ERC-20 代币，这意味着调用的 ERC-20 合约可能会导致调用合约的 Uniswap 市场遭受攻击。 使用 `unlocked` 变量， 我们可以防止函数在运行时被调用(在相同的交易内)。

```solidity
    modifier lock() {
```

此函数是一个 [modifier](https://docs.soliditylang.org/en/v0.8.3/contracts.html#function-modifiers) 函数，用于以某种方式改变正常函数的行为。

```solidity
        require(unlocked == 1, 'UniswapV2: LOCKED');
        unlocked = 0;
```

如果 `unlocked` 变量值为 1，将其设置为 0。 如果已经是 0，则撤销调用，返回失败。

```solidity
        _;
```

在修饰符中，`_;` 是原始函数调用（含所有参数）。 这里表明仅在 `unlocked` 变量值为 1 时 才能调用函数，而当函数运行时，`unlocked` 值为 0。

```solidity
        unlocked = 1;
    }
```

当主函数返回后，释放锁定。

**其他 函数**

```solidity
    function getReserves() public view returns (uint112 _reserve0, uint112 _reserve1, uint32 _blockTimestampLast) {
        _reserve0 = reserve0;
        _reserve1 = reserve1;
        _blockTimestampLast = blockTimestampLast;
    }
```

此函数返回给调用者当前的兑换状态。 请注意，Solidity 函数[可以返回多个 值](https://docs.soliditylang.org/en/v0.8.3/contracts.html#returning-multiple-values)。

```solidity
    function _safeTransfer(address token, address to, uint value) private {
        (bool success, bytes memory data) = token.call(abi.encodeWithSelector(SELECTOR, to, value));
```

此内部函数可以从交易所转账一定数额的 ERC20 代币给其他账户。 `SELECTOR` 指定 我们调用的函数是 `transfer(address,uint)`（参见上面的定义）。

为了避免必须为代币函数导入接口，我们需要使用其中一个 [ABI 函数](https://docs.soliditylang.org/en/v0.8.3/units-and-global-variables.html#abi-encoding-and-decoding-functions) 来“手动”创建调用。

```solidity
        require(success && (data.length == 0 || abi.decode(data, (bool))), 'UniswapV2: TRANSFER_FAILED');
    }
```

ERC-20 的转移调用有两种方式可能失败：

1. 回滚 如果对外部合约的调用回滚，则布尔返回值为 `false`
2. 正常结束但报告失败。 在这种情况下，返回值的缓冲为非零长度，将其解码为布尔值时，其值为 `false`

一旦出现这两种情况，转移调用就会回退。

**事件**

```solidity
    event Mint(address indexed sender, uint amount0, uint amount1);
    event Burn(address indexed sender, uint amount0, uint amount1, address indexed to);
```

当流动资金提供者存入流动资金 (`Mint`) 或提取流动资金 (`Burn`) 时，会发出这两个事件。 在 这两种情况下，存入或提取出的 token0 和 token1 的金额是事件的一部分， 以及调用合约的账户地址 (`Sender`)。 在提取资金时，事件中还包括获得代币的目标地址 (`to`) 这个地址可能与发送合约的账户地址不同。

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

当交易者用一种代币交换另一种代币时，会激发此事件。 同样，代币发送者和兑换后代币的存入目的账户可能不一样。 每种代币都可以发送到交易所，或者从交易所接收。

```solidity
    event Sync(uint112 reserve0, uint112 reserve1);
```

最后，每次存入或提取代币时都会发出 `Sync`，无论出于何种原因，提供最新的储备信息 （从而提供汇率）。

**设置函数**

这些函数应在建立新的配对交易时调用。

```solidity
    constructor() public {
        factory = msg.sender;
    }
```

构造函数确保我们能够跟踪产生配对的工厂合约的地址。 `initialize` 函数和工厂合约执行费（如果有）需要此信息

```solidity
    // called once by the factory at time of deployment
    function initialize(address _token0, address _token1) external {
        require(msg.sender == factory, 'UniswapV2: FORBIDDEN'); // sufficient check
        token0 = _token0;
        token1 = _token1;
    }
```

这个函数允许工厂（而且只允许工厂）指定配对中进行兑换的两种 ERC-20 代币。

**内部更新函数**

**\_update**

```solidity
    // update reserves and, on the first call per block, price accumulators
    function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
```

每次存入或提取代币时，会调用此函数。

```solidity
        require(balance0 <= uint112(-1) && balance1 <= uint112(-1), 'UniswapV2: OVERFLOW');
```

如果 balance0 或 balance1 (uint256) 高于 uint112(-1) (=2^112-1)（因此当转换为 uint112 时会溢出并返回 0) 拒绝 继续 \_update 以防止溢出。 一般的代币可以细分成 10^18 个单元，这意味着 代币每次的兑换限制大约为每个代币的 5.1\*10^15。 迄今为止，这并不是一个问题。

```solidity
        uint32 blockTimestamp = uint32(block.timestamp % 2**32);
        uint32 timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
        if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
```

如果流逝的时间值不是零，这意味着本交易是此区块上的第一笔兑换交易。 在这种情况下，我们需要更新累积成本值。

```solidity
            // * never overflows, and + overflow is desired
            price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
            price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
        }
```

每个累积成本值都用最新成本值（另一个代币的储备金额/本代币的储备金额）乘以以秒为单位的流逝时间加以更新。 要获得平均兑换价格，需要读取两个累积成本值，并除以它们之间的时间差。 例如，假设下面这些事件序列：

| 事件                                 |  reserve0 |  reserve1 | 时间戳   | 边际汇率 (reserve1 / reserve0) |       price0CumulativeLast |
| ---------------------------------- | --------: | --------: | ----- | -------------------------: | -------------------------: |
| 初始设置                               | 1,000.000 | 1,000.000 | 5,000 |                      1.000 |                          0 |
| 交易者 A 存入 50 个代币 0 获得 47.619 个代币 1  | 1,050.000 |   952.381 | 5,020 |                      0.907 |                         20 |
| 交易者 B 存入 10 个代币 0 获得 8.984 个代币 1   | 1,060.000 |   943.396 | 5,030 |                       0.89 |       20+10\*0.907 = 29.07 |
| 交易者 C 存入 40 个代币 0 获得 34.305 个代币 1  | 1,100.000 |   909.090 | 5,100 |                      0.826 |    29.07+70\*0.890 = 91.37 |
| 交易者 D 存入 100 个代币 0 获得 109.01 个代币 1 |   990.990 | 1,009.090 | 5,110 |                      1.018 |    91.37+10\*0.826 = 99.63 |
| 交易者 E 存入 10 个代币 0 获得 10.079 个代币 1  | 1,000.990 |   999.010 | 5,150 |                      0.998 | 99.63+40\*1.1018 = 143.702 |

比如说我们想要计算时间戳 5,030 到 5,150 之间**代币 0** 的平均价格。 `price0Cumulative` 的差值 为 143.702-29.07=114.632。 此为两分钟（120 秒）间的平均值。 因此，平均价格为 114.632/120 = 0.955。

此价格计算是我们需要知道原有资金储备规模的原因。

```solidity
        reserve0 = uint112(balance0);
        reserve1 = uint112(balance1);
        blockTimestampLast = blockTimestamp;
        emit Sync(reserve0, reserve1);
    }
```

最后，更新全局变量并发布一个 `Sync` 事件。

**\_mintFee**

```solidity
    // if fee is on, mint liquidity equivalent to 1/6th of the growth in sqrt(k)
    function _mintFee(uint112 _reserve0, uint112 _reserve1) private returns (bool feeOn) {
```

在 Uniswap 2.0 的合约中规定交易者为使用兑换市场支付 0.30% 的费用。 这笔费用的大部分（交易的 0.25%）支付给流动资金提供者。 余下的 0.5% 可以支付给流动资金提供者或由工厂合约指定的账户地址作为协议费，可以用于支付 Uniswap 团队的开发费用。

为了减少计算次数（因此减少燃料费用），这笔费用只在流动资金被添加或移除时才计算，而不是在每次兑换交易时计算。

```solidity
        address feeTo = IUniswapV2Factory(factory).feeTo();
        feeOn = feeTo != address(0);
```

读取工厂的费用支付地址。 如果返回值为零，则代表没有协议费， 也不需要来计算这笔费用。

```solidity
        uint _kLast = kLast; // gas savings
```

`kLast` 状态变量位于内存中，所以在合约的不同调用中都有一个值。 虽然函数内存每次在函数调用后都会清空，但由于访问存储的费用要比访问内存要高得多， 所以我们使用内存的内部变量来代表存储变量的值，以降低燃料费用。

```solidity
        if (feeOn) {
            if (_kLast != 0) {
```

流动资金提供者仅仅因为提供流动性代币而得到所属的费用。 但是协议 费用要求发行新的流动性代币，并提供给 `feeTo` 的账户地址。

```solidity
                uint rootK = Math.sqrt(uint(_reserve0).mul(_reserve1));
                uint rootKLast = Math.sqrt(_kLast);
                if (rootK > rootKLast) {
```

如果有新的流动性变化需要收取协议费。 你可以在 [本文后面](uniswapv2-contracts-guide.md#Math)看到平方根函数。

```solidity
                    uint numerator = totalSupply.mul(rootK.sub(rootKLast));
                    uint denominator = rootK.mul(5).add(rootKLast);
                    uint liquidity = numerator / denominator;
```

这种复杂的费用计算方法在[白皮书](https://uniswap.org/whitepaper.pdf)第 5 页中作了解释。 在计算 `kLast` 的间隔期间，流动性没有变化（因为每次计算 都是在流动性发生实际变化时发生），所以 `reserve0 * reserve1` 的变化 一定是从交易费用中产生（没有交易费用的话 `reserve0 * reserve1` 值为常量）。

```solidity
                    if (liquidity > 0) _mint(feeTo, liquidity);
                }
            }
```

使用 `UniswapV2ERC20._mint` 函数产生更多的流动池代币并发送到 `feeTo` 地址。

```solidity
        } else if (_kLast != 0) {
            kLast = 0;
        }
    }
```

如果不需收费则将 `klast` 设为 0（如果 klast 不为 0）。 编写该合约时，有一个[燃料返还功能](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-3298.md)，用于鼓励合约将其不需要的存储释放，从而减少以太坊上状态变量的整体存储大小。 此段代码在可行时返还。

**外部可访问函数**

请注意，虽然这些函数\_可以\_被任意交易或合约调用，其设计目的是用于外部合约调用。 如果直接调用，您无法骗过配对交易， 可能因错误而丢失价值。

**铸币**

```solidity
    // this low-level function should be called from a contract which performs important safety checks
    function mint(address to) external lock returns (uint liquidity) {
```

当流动资金提供者为资金池增加流动资金时，将会调用此函数。 它将产生额外的流动池 代币作为奖励。 在 [外围合约](uniswapv2-contracts-guide.md#UniswapV2Router02)中增加流动性后调用这个函数，以确保二者在同一交易中（因此其他人都不能提交向合法所有者要求新流动资金的交易）。

```solidity
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
```

这是 Solidity 函数中读取多个返回值的方式。 这里我们忽略了最后 返回的值，即区块时间戳，因为不需要它。

```solidity
        uint balance0 = IERC20(token0).balanceOf(address(this));
        uint balance1 = IERC20(token1).balanceOf(address(this));
        uint amount0 = balance0.sub(_reserve0);
        uint amount1 = balance1.sub(_reserve1);
```

获取当前余额并查看每个代币类型中添加的数量。

```solidity
        bool feeOn = _mintFee(_reserve0, _reserve1);
```

如果有协议费用的话，计算需要收取的费用，并相应地产生流动池代币。 因为输入 `_mintFee` 函数的参数是原有的储备价值，相应费用的计算只是基于费用 导致的流动池变化。

```solidity
        uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
        if (_totalSupply == 0) {
            liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
           _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
```

如果这是第一笔存款，会创建数量为 `MINIMUM_LIQUIDITY` 的代币并将它们发送到地址 0 进行锁定。 这些代币 无法被获取，也就是说流动池永远不会为空（避免之后的计算中 出现除零错误）。 `MINIMUM_LIQUIDITY` 的值是 1000，因为考虑到大多数 ERC-20 被细分成 1 个代币的 10^-18 个单位，而以太币则被分为 wei，为 1 个代币价值的 10^-15。 成本不高。

在首次存款时，我们不知道两个代币的相对价值，所以假定两种代币都具有相同的价值，只需要两者数量的乘积并取一下方根。

我们可以相信这一点，因为提供同等价值、避免套利符合存款人的利益。 比方说，这两种代币的价值是相同的，但我们的存款人存入的 **Token1** 是 **Token0** 的四倍。 通过配对交易，交易者可以认为 **Token0** 的价值 比较高。

| 事件                                            | reserve0 | reserve1 | reserve0 \* reserve1 | 流动池价值 (reserve0 + reserve1) |
| --------------------------------------------- | -------: | -------: | -------------------: | --------------------------: |
| 初始设置                                          |        8 |       32 |                  256 |                          40 |
| 交易者存入 8 个 **Token0** 代币，获得 16 个 **Token1** 代币 |       16 |       16 |                  256 |                          32 |

正如您可以看到的，交易者额外获得了 8 个代币，这是由于流动池价值下降造成的，损害了拥有流动池的存款人。

```solidity
        } else {
            liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);
```

对于随后每一笔存款，我们都知道了两种资产之间的汇率。我们期望流动资金提供者提供 等值的两种代币。 如果他们没有，我们根据他们提供的较低价值代币来支付他们的流动池代币以做惩罚。

无论是最初存入还是后续存入，流动池的代币金额均等于 `reserve0*reserve1` 的 平方根，而流动池代币的价值不变（除非存入的资金为不等值的代币类型， 那么就会分派“罚金”）。 这里有另一个例子，两种代币具有相同价值，有三个良性的存款和一个恶性的存款 （即只存入一种类型的代币，所以不会产生任何流动池代币）。

| 事件         | reserve0 | reserve1 | reserve0 \* reserve1 | 流动池价值 (reserve0 + reserve1) | 存入资金而产生的流动池代币 | 流动池代币总值 | 每个流动池代币的值 |
| ---------- | -------: | -------: | -------------------: | --------------------------: | ------------: | ------: | --------: |
| 初始设置       |    8.000 |    8.000 |                   64 |                      16.000 |             8 |       8 |     2.000 |
| 每种代币存入 4 个 |   12.000 |   12.000 |                  144 |                      24.000 |             4 |      12 |     2.000 |
| 每种代币存入 2 个 |   14.000 |   14.000 |                  196 |                      28.000 |             2 |      14 |     2.000 |
| 不等值的存款     |   18.000 |   14.000 |                  252 |                      32.000 |             0 |      14 |   \~2.286 |
| 套利后        | \~15.874 | \~15.874 |                  252 |                    \~31.748 |             0 |      14 |   \~2.267 |

```solidity
        }
        require(liquidity > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_MINTED');
        _mint(to, liquidity);
```

使用 `UniswapV2ERC20._mint` 函数产生更多流动池代币并发送到正确的账户地址。

```solidity
        _update(balance0, balance1, _reserve0, _reserve1);
        if (feeOn) kLast = uint(reserve0).mul(reserve1); // reserve0 and reserve1 are up-to-date
        emit Mint(msg.sender, amount0, amount1);
    }
```

更新相应的状态变量（`reserve0`、`reserve1`，必要时还包含 `kLast`）并激发相应事件。

**销毁**

```solidity
    // this low-level function should be called from a contract which performs important safety checks
    function burn(address to) external lock returns (uint amount0, uint amount1) {
```

当流动资金被提取且相应的流动池代币需要被销毁时，将调用此函数。 还需要[从一个外围账户](uniswapv2-contracts-guide.md#UniswapV2Router02)调用。

```solidity
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
        address _token0 = token0;                                // gas savings
        address _token1 = token1;                                // gas savings
        uint balance0 = IERC20(_token0).balanceOf(address(this));
        uint balance1 = IERC20(_token1).balanceOf(address(this));
        uint liquidity = balanceOf[address(this)];
```

外围合约在调用函数之前，首先将要销毁的流动资金转到本合约中。 这样 我们知道有多少流动资金需要销毁，并可以确保它被销毁。

```solidity
        bool feeOn = _mintFee(_reserve0, _reserve1);
        uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
        amount0 = liquidity.mul(balance0) / _totalSupply; // using balances ensures pro-rata distribution
        amount1 = liquidity.mul(balance1) / _totalSupply; // using balances ensures pro-rata distribution
        require(amount0 > 0 && amount1 > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_BURNED');
```

流动资金提供者获得等值数量的两种代币。 这样不会改变兑换汇率。

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

`burn` 函数的其余部分是上述 `mint` 函数的镜像。

**兑换**

```solidity
    // this low-level function should be called from a contract which performs important safety checks
    function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external lock {
```

此函数也应该从[外围合约](uniswapv2-contracts-guide.md#UniswapV2Router02)调用。

```solidity
        require(amount0Out > 0 || amount1Out > 0, 'UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT');
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
        require(amount0Out < _reserve0 && amount1Out < _reserve1, 'UniswapV2: INSUFFICIENT_LIQUIDITY');

        uint balance0;
        uint balance1;
        { // scope for _token{0,1}, avoids stack too deep errors
```

本地变量可以存储在内存中，或者如果变量数目不太多，直接存储进堆栈。 如果我们可以限制变量数量，那么建议使用堆栈以减少燃料消耗。 更多详情见 [以太坊黄皮书](https://ethereum.github.io/yellowpaper/paper.pdf)（以前的以太坊规范）p. 26“方程式 298”。

```solidity
            address _token0 = token0;
            address _token1 = token1;
            require(to != _token0 && to != _token1, 'UniswapV2: INVALID_TO');
            if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out); // optimistically transfer tokens
            if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out); // optimistically transfer tokens
```

这种转移应该是会成功的，因为在转移之前我们确信所有条件都得到满足。 在以太坊中这样操作是可以的， 原因在于如果调用条件没有得到满足，我们可以恢复操作及造成的改变。

```solidity
            if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);
```

如果收到请求，则通知接收者要进行兑换。

```solidity
            balance0 = IERC20(_token0).balanceOf(address(this));
            balance1 = IERC20(_token1).balanceOf(address(this));
        }
```

获取当前余额。 外围合约在调用交换函数之前，需要向合约发送要兑换的代币。 这个功能可以使得合约检查它没有受到欺骗，这个检查\_必须\_通过核心合约调用（因为本功能可能被除我们外围合约之外的其它单位调用）。

```solidity
        uint amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
        uint amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
        require(amount0In > 0 || amount1In > 0, 'UniswapV2: INSUFFICIENT_INPUT_AMOUNT');
        { // scope for reserve{0,1}Adjusted, avoids stack too deep errors
            uint balance0Adjusted = balance0.mul(1000).sub(amount0In.mul(3));
            uint balance1Adjusted = balance1.mul(1000).sub(amount1In.mul(3));
            require(balance0Adjusted.mul(balance1Adjusted) >= uint(_reserve0).mul(_reserve1).mul(1000**2), 'UniswapV2: K');
```

这是一项健全性检查，确保我们不会因兑换而损失代币。 在任何情况下交换都不应减少 `reserve0*reserve1`。 这也是我们确保为兑换发送 0.3% 费用的方式；在对 K 值进行完整性检查之前，我们将两个余额乘以 1000 减去 3 倍的金额，这意味着在将其 K 值与当前准备金 K 值进行比较之前，从余额中扣除 0.3% (3/1000 = 0.003 = 0.3%)。

```solidity
        }

        _update(balance0, balance1, _reserve0, _reserve1);
        emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
    }
```

更新 `reserve0` 和 `reserve1` 的值，并在必要时更新价格累积值和时间戳并激发相应事件。

**同步或提取**

实际余额有可能与配对交易所认为的储备金余额没有同步。 没有合约的认同，就无法撤回代币，但存款却不同。 帐户 可以将代币转移到交易所，而无需调用 `mint` 或 `swap`。

在这种情况下，有两种解决办法：

* `sync`，将储备金更新为当前余额
* `skim`，撤回额外的金额。 请注意任何账户都可以调用 `skim` 函数，因为无法知道是谁 存入的代币。 此信息是在一个事件中发布的，但这些事件无法从区块链中访问。

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

#### UniswapV2Factory.sol <a href="#uniswapv2factory" id="uniswapv2factory"></a>

[此合约](https://github.com/Uniswap/uniswap-v2-core/blob/master/contracts/UniswapV2Factory.sol)实现配对 兑换。

```solidity
pragma solidity =0.5.16;

import './interfaces/IUniswapV2Factory.sol';
import './UniswapV2Pair.sol';

contract UniswapV2Factory is IUniswapV2Factory {
    address public feeTo;
    address public feeToSetter;
```

这些状态变量是执行协议费用所必需的（请见[白皮书](https://uniswap.org/whitepaper.pdf)的第 5 页）。 `feeTo` 地址用于累加协议费用的流动池代币，而 `feeToSetter` 是允许更改 `feeTo` 为 不同地址的地址值。

```solidity
    mapping(address => mapping(address => address)) public getPair;
    address[] public allPairs;
```

这些变量用以跟踪配对，即两种代币之间的兑换。

第一个变量，`getPair` 是一个映射，根据兑换的两个 ERC-20 代币 来识别配对交易合约。 ERC-20 代币通过实现合约的地址来识别，所以关键字和值都是地址。 为了获取 配对交易的地址，以便能够从 `tokenA` 转换为 `tokenB`，可以使用 `getPair [<tokenA address><tokenB address>]`（或反之）。

第二个变量，`allPairs` 是一个数组，其中包括该工厂创建的所有 配对交易的地址。 在以太坊中，您无法循环访问映射内容， 或获取所有关键字的列表，所以，这个变量是唯一能够知道此工厂 管理哪个兑换的方法。

注意: 您不能循环访问所有关键字的原因是合约数据 存储\_十分昂贵\_，所以我们越少用越好，且越少改变 越好。 您可以创建[支持循环访问的映射](https://github.com/ethereum/dapp-bin/blob/master/library/iterable\_mapping.sol)， 但它们需要额外存储关键字列表。 但在大多数应用程序中并不需要。

```solidity
    event PairCreated(address indexed token0, address indexed token1, address pair, uint);
```

当新的配对交易创建时，将激发此事件。 它包括代币地址、 配对交易地址以及工厂管理的兑换交易总数。

```solidity
    constructor(address _feeToSetter) public {
        feeToSetter = _feeToSetter;
    }
```

构造函数做的唯一事情是指定 `feeToSetter`。 工厂开始时没有 费用，只有 `feeSetter` 可以更改这种情况。

```solidity
    function allPairsLength() external view returns (uint) {
        return allPairs.length;
    }
```

此函数返回交易配对的数量。

```solidity
    function createPair(address tokenA, address tokenB) external returns (address pair) {
```

这是工厂的主要函数，可以在两个 ERC-20 代币之间创建配对交易。 注意， 任何人都可以调用此函数。 并不需要 Uniswap 许可就能创建新的配对 兑换。

```solidity
        require(tokenA != tokenB, 'UniswapV2: IDENTICAL_ADDRESSES');
        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
```

我们希望新兑换的地址可以确定， 这样它可以在链下预计算 （这对于[第二层的交易](../developers/docs/layer-2-scaling/) 来说比较有用）。 为了做到这一点，我们需要代币地址始终按顺序排列，无论收到代币地址的顺序如何， 都需要在这里排序。

```solidity
        require(token0 != address(0), 'UniswapV2: ZERO_ADDRESS');
        require(getPair[token0][token1] == address(0), 'UniswapV2: PAIR_EXISTS'); // single check is sufficient
```

大流动资金池优于小流动资金池，因为其价格比较稳定。 对于每一对代币， 我们不想有多个流动资金池。 如果已经有一个配对交易，则无需为相同的代币对 创建另一个配对交易。

```solidity
        bytes memory bytecode = type(UniswapV2Pair).creationCode;
```

为了创建一个新的合约，我们需要获得创建代码（包括构造函数和写入 用于存储实际合约以太坊虚拟机字节码的代码）。 在 Solidity 语言中，通常使用 `addr = new <name of contract>(<constructor parameters>)` 的格式语句，然后编译器就可以完成所有的工作，不过为了获取一个确定的合约地址，需要使用 [CREATE2 操作码](https://eips.ethereum.org/EIPS/eip-1014)。 当这个代码编写出来时，Solidity 还不支持操作码，因此需要手动获取 代码。 目前这已经不再是问题，因为 [Solidity 现已支持 CREATE2](https://docs.soliditylang.org/en/v0.8.3/control-structures.html#salted-contract-creations-create2)。

```solidity
        bytes32 salt = keccak256(abi.encodePacked(token0, token1));
        assembly {
            pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
        }
```

当 Solidity 不支持操作码时，我们可以通过[内联汇编](https://docs.soliditylang.org/en/v0.8.3/assembly.html)来调用。

```solidity
        IUniswapV2Pair(pair).initialize(token0, token1);
```

调用 `initialize` 函数来告诉新兑换交易可以兑换哪两种代币。

```solidity
        getPair[token0][token1] = pair;
        getPair[token1][token0] = pair; // populate mapping in the reverse direction
        allPairs.push(pair);
        emit PairCreated(token0, token1, pair, allPairs.length);
    }
```

在状态变量中保存新的配对信息，并激发一个事件来告知外界新的配对交易合约已生成。

```solidity
    function setFeeTo(address _feeTo) external {
        require(msg.sender == feeToSetter, 'UniswapV2: FORBIDDEN');
        feeTo = _feeTo;
    }

    function setFeeToSetter(address _feeToSetter) external {
        require(msg.sender == feeToSetter, 'UniswapV2: FORBIDDEN');
        feeToSetter = _feeToSetter;
    }
}
```

这两个函数，允许 `setFeeTo` 管理费用的接收者（如有），并将 `setFeeToSetter` 更改为一个新 地址。

#### UniswapV2ERC20.sol <a href="#uniswapv2erc20" id="uniswapv2erc20"></a>

[本合约](https://github.com/Uniswap/uniswap-v2-core/blob/master/contracts/UniswapV2ERC20.sol)实现了 ERC-20 流动代币。 这与 [OpenWhisk ERC-20 合约](../developers/tutorials/erc20-annotated-code/)相似，因此 这里仅解释不同的部分，`permit` 的功能。

以太坊上的交易需要消耗以太币 (ETH)，相当于实际货币。 如果您有 ERC-20 代币但没有以太币，就无法发送 交易，因而不能用代币做任何事情。 避免该问题的一个解决方案是 [元交易](https://docs.uniswap.org/protocol/V2/guides/smart-contract-integration/supporting-meta-transactions/)。 代币的所有者签署一个交易，许可他人将代币从链上取出，并通过网络将其发送给 接收人。 接收人拥有以太币，可以代表所有者提交许可。

```solidity
    bytes32 public DOMAIN_SEPARATOR;
    // keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)");
    bytes32 public constant PERMIT_TYPEHASH = 0x6e71edae12b1b97f4d1f60370fef10105fa2faae0126114a169c64845d6126c9;
```

此哈希值是[这种交易类型的标识](https://eips.ethereum.org/EIPS/eip-712#rationale-for-typehash)。 在这里 我们唯一支持的是带有这些参数的 `Permit`。

```solidity
    mapping(address => uint) public nonces;
```

接收人无法伪造数字签名。 但是，可以两次发送相同的交易 （这是一种[重放攻击](https://wikipedia.org/wiki/Replay\_attack)形式）。 为防止这种情况，我们使用 一个[随机数](https://wikipedia.org/wiki/Cryptographic\_nonce)。 如果新 `Permit` 的随机数不是上一次的使用的随机数加一， 我们便判定它无效。

```solidity
    constructor() public {
        uint chainId;
        assembly {
            chainId := chainid
        }
```

这是获取[链标识符](https://chainid.network/)的代码。 它使用名为 [Yul](https://docs.soliditylang.org/en/v0.8.4/yul.html) 的以太坊虚拟机编译语言。 请注意，在当前版本 Yul 中，您必须使用 `chainid()`， 而非 `chainid`。

```solidity
        DOMAIN_SEPARATOR = keccak256(
            abi.encode(
                keccak256('EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)'),
                keccak256(bytes(name)),
                keccak256(bytes('1')),
                chainId,
                address(this)
            )
        );
    }
```

计算 EIP-712 的[域分隔符](https://eips.ethereum.org/EIPS/eip-712#rationale-for-domainseparator)。

```solidity
    function permit(address owner, address spender, uint value, uint deadline, uint8 v, bytes32 r, bytes32 s) external {
```

这是实现批准功能的函数。 它接收相关字段的参数，以及[数字签名](https://yos.io/2018/11/16/ethereum-signatures/) 的三个标量值（v、r 和 s）。

```solidity
        require(deadline >= block.timestamp, 'UniswapV2: EXPIRED');
```

截止日期后请勿接受交易。

```solidity
        bytes32 digest = keccak256(
            abi.encodePacked(
                '\x19\x01',
                DOMAIN_SEPARATOR,
                keccak256(abi.encode(PERMIT_TYPEHASH, owner, spender, value, nonces[owner]++, deadline))
            )
        );
```

`abi.encodePacked(...)` 是我们预计将收到的信息。 我们知道随机数应该是什么，所以不需要 将它作为一个参数

以太坊签名算法预计获得 256 位用于签名，所以我们使用 `keccak256` 哈希函数。

```solidity
        address recoveredAddress = ecrecover(digest, v, r, s);
```

从摘要和签名中，我们可以用 [ecrecover](https://coders-errand.com/ecrecover-signature-verification-ethereum/) 函数计算出签名的地址。

```solidity
        require(recoveredAddress != address(0) && recoveredAddress == owner, 'UniswapV2: INVALID_SIGNATURE');
        _approve(owner, spender, value);
    }
```

如果一切正常，则将其视为 [ERC-20 批准](https://eips.ethereum.org/EIPS/eip-20#approve)。

### 外围合约 <a href="#periphery-contracts" id="periphery-contracts"></a>

外围合约是用于 Uniswap 的 API（应用程序接口）。 它们可用于来自 其他合约或去中心化应用程序的外部调用。 你可以直接调用核心合约，但这更为复杂， 如果您犯了错误，则可能会丢失值。 核心合约只包含确保它们不会遭受欺骗的测试，不会对其他调用者进行健全性检查。 它们在外围，因此可以根据需要进行更新。

#### UniswapV2Router01.sol <a href="#uniswapv2router01" id="uniswapv2router01"></a>

[该合约](https://github.com/Uniswap/uniswap-v2-periphery/blob/master/contracts/UniswapV2Router01.sol) 存有问题，[不应该再使用](https://docs.uniswap.org/protocol/V2/reference/smart-contracts/router-01/)。 幸运的是， 外围合约无状态记录，也不拥有任何资产，所以很容易废弃。建议 使用 `UniswapV2Router02` 来替代。

#### UniswapV2Router02.sol <a href="#uniswapv2router02" id="uniswapv2router02"></a>

在大多数情况下，您会通过[该合约](https://github.com/Uniswap/uniswap-v2-periphery/blob/master/contracts/UniswapV2Router02.sol)使用 Uniswap。 有关使用说明，您可以在[这里](https://docs.uniswap.org/protocol/V2/reference/smart-contracts/router-02/)找到。

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

其中大部分我们都曾遇到过，或相当明显。 一个例外是 `IWETH.sol`。 Uniswapv2 允许兑换 任意一对 ERC-20 代币，但以太币 (ETH) 本身并不是 ERC-20 代币。 它早于该标准出现，并采用独特的机制转换。 为了 在适用于 ERC-20 代币的合约中使用以太币，人们制定出[包装以太币 (WETH)](https://weth.io/) 合约。 您 发送以太币到该合约，它会为您铸造相同金额的包装以太币。 或者您可以销毁包装以太币，然后换回以太币。

```solidity
contract UniswapV2Router02 is IUniswapV2Router02 {
    using SafeMath for uint;

    address public immutable override factory;
    address public immutable override WETH;
```

路由需要知道使用哪个工厂，以及对于需要包装以太币的交易，要使用什么包装以太币合约。 这些变量值是 [不可修改](https://docs.soliditylang.org/en/v0.8.3/contracts.html#constant-and-immutable-state-variables)的，意味着它们 只能在构造函数中设置。 这使得用户可以相信没有人能够改变它们，比如指向有风险 的合约。

```solidity
    modifier ensure(uint deadline) {
        require(deadline >= block.timestamp, 'UniswapV2Router: EXPIRED');
        _;
    }
```

此修改函数确保有时间限制的交易（如果可以，请在 Y 之前执行 X）不会在时限后发生。

```solidity
    constructor(address _factory, address _WETH) public {
        factory = _factory;
        WETH = _WETH;
    }
```

构造函数仅用于设置不可变的状态变量。

```solidity
    receive() external payable {
        assert(msg.sender == WETH); // only accept ETH via fallback from the WETH contract
    }
```

当我们将代币从包装以太币合约换回以太币时，需要调用此函数。 只有我们使用的包装以太币合约才能授权 完成此操作。

**增加流动资金**

这些函数添加代币进行配对交易，从而增大了流动资金池。

```solidity
    // **** ADD LIQUIDITY ****
    function _addLiquidity(
```

此函数用于计算应存入 配对交易的 A 代币和 B 代币的金额。

```solidity
        address tokenA,
        address tokenB,
```

这些是 ERC-20 代币合约的地址。

```solidity
        uint amountADesired,
        uint amountBDesired,
```

这些是流动资金提供者想要存入的代币数额。 它们也是要存入的 A 和 B 的最大数额。

```solidity
        uint amountAMin,
        uint amountBMin
```

这些是可接受的最低存款数额。 如果大于这些金额的交易无法完成， 则会回退。 如果不想要此功能，将它们设定为零即可。

流动资金提供者指定最低限额的目的，是想要将交易限制在 与当前汇率接近的汇率。 如果汇率波动太大， 可能意味着基础价值可能发生改变，他们需要人工决定做什么。

例如，想象汇率是一比一时，流动资金提供者 指定了这些值：

| 参数             |    值 |
| -------------- | ---: |
| amountADesired | 1000 |
| amountBDesired | 1000 |
| amountAMin     |  900 |
| amountBMin     |  800 |

只要汇率保持在 0.9 至 1.25 之间，交易就会进行。 如果汇率超出这个范围，交易将被取消。

采取这种预防措施的原因是交易不是立即执行的，提交这些交易之后，最终 要等到矿工会将它们包含在区块中才算执行完（除非交易的燃料价格非常低，在这种情况下， 需要提交另一笔具有相同随机数和更高燃料价格的交易，以覆盖前一笔交易）。 在 提交交易和写入区块之间发生的事情是无法控制的。

```solidity
    ) internal virtual returns (uint amountA, uint amountB) {
```

该函数返回流动资金提供者应存入的金额，其比率等于当前 储备金之间的比率。

```solidity
        // create the pair if it doesn't exist yet
        if (IUniswapV2Factory(factory).getPair(tokenA, tokenB) == address(0)) {
            IUniswapV2Factory(factory).createPair(tokenA, tokenB);
        }
```

如果还没有此代币对的兑换交易，则创建一个。

```solidity
        (uint reserveA, uint reserveB) = UniswapV2Library.getReserves(factory, tokenA, tokenB);
```

获取配对中的当前储备金。

```solidity
        if (reserveA == 0 && reserveB == 0) {
            (amountA, amountB) = (amountADesired, amountBDesired);
```

如果当前储备金为空，那么这是一笔新的配对交易。 存入的金额应与 流动资金提供者想要提供的金额完全相同。

```solidity
        } else {
            uint amountBOptimal = UniswapV2Library.quote(amountADesired, reserveA, reserveB);
```

如果我们需要知道要多大的金额，我们可使用 [此函数](https://github.com/Uniswap/uniswap-v2-periphery/blob/master/contracts/libraries/UniswapV2Library.sol#L35)获得最佳金额。 我们想要与当前储备相同的比率。

```solidity
            if (amountBOptimal <= amountBDesired) {
                require(amountBOptimal >= amountBMin, 'UniswapV2Router: INSUFFICIENT_B_AMOUNT');
                (amountA, amountB) = (amountADesired, amountBOptimal);
```

如果最佳金额 `amountBOptimal` 小于流动资金提供者想要存入的金额，这意味着代币 B 目前比流动资金存款人所认为的价值更高，所以需要更少的数额。

```solidity
            } else {
                uint amountAOptimal = UniswapV2Library.quote(amountBDesired, reserveB, reserveA);
                assert(amountAOptimal <= amountADesired);
                require(amountAOptimal >= amountAMin, 'UniswapV2Router: INSUFFICIENT_A_AMOUNT');
                (amountA, amountB) = (amountAOptimal, amountBDesired);
```

如果 B 代币的最佳数额大于所需的 B 代币数额，这意味着 B 代币目前的价值 低于流动资金存款人的估计，所以需要更高的金额。 然而，所需的金额是最大值，意味着我们无法存入更多数量的 B 代币。 可以选择的另一种方法是，我们计算所需 B 代币数额对应的最佳 A 代币数额。

把数值汇总起来，我们就会得到这张图表。 假定您正在试图存入 1000 个 A 代币（蓝线）和 1000 个 B 代币（红线）。 X 轴是汇率，A/B。 如果 x=1，它们的价值相等，并且你每次可以存入 1000 个 A 代币和 1000 个 B 代币。 如果 x=2，A 的价值是 B 的两倍（每个 A 代币可换两个 B 代币），所以您可以存 1000 个 B 代币， 但只能存 500 个 A 代币。 如果是 x=0.5，情况就会逆转，即可存 1000 个 A 代币或 500 个 B 代币。

![图表](../block%20chain/liquidityProviderDeposit.png)

```solidity
            }
        }
    }
```

您可以将流动资金直接存入核心合约（使用 [UniswapV2Pair:::mint](https://github.com/Uniswap/uniswap-v2-core/blob/master/contracts/UniswapV2Pair.sol#L110)），但核心合约 只是检查合约自己没有遭受欺骗。因此，如果汇率在 提交交易至执行交易之间发生变化，您将面临损失资金价值的风险。 如果使用外围合约，它会计算您应该存入 的金额并会立即存入，所以汇率不会改变，您不会损失资金价值。

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

此函数可以在交易中调用，用于存入流动资金。 大多数参数与上述 `_addLiquidity` 中相同，但有两个例外：

. `to` 是会获取新流动池代币的地址，这些代币铸造用于显示流动资金提供者在池中所占比率 `deadline` 是交易的时间限制

```solidity
    ) external virtual override ensure(deadline) returns (uint amountA, uint amountB, uint liquidity) {
        (amountA, amountB) = _addLiquidity(tokenA, tokenB, amountADesired, amountBDesired, amountAMin, amountBMin);
        address pair = UniswapV2Library.pairFor(factory, tokenA, tokenB);
```

我们计算实际存入的金额，然后找到流动资金池的账户地址。 为了节省燃料，我们不用 询问工厂，但可以使用库函数 `pairFor`（参见如下程序库）

```solidity
        TransferHelper.safeTransferFrom(tokenA, msg.sender, pair, amountA);
        TransferHelper.safeTransferFrom(tokenB, msg.sender, pair, amountB);
```

将正确数额的代币从用户账户转到配对交易。

```solidity
        liquidity = IUniswapV2Pair(pair).mint(to);
    }
```

反过来，将流动资金池的部分所有权赋予 `to` 地址的流动性代币。 核心 合约的 `mint` 函数可以查看到它有多少额外的代币（与 上次流动性发生变化时所有的数额进行比较），并相应地铸造流动性代币。

```solidity
    function addLiquidityETH(
        address token,
        uint amountTokenDesired,
```

当流动资金提供者想要向代币/以太币配对交易提供流动资金时，存在一些差别。 合约 为流动资金提供者处理以太币的包装。 用户不需要指定想要存入多少以太币， 因为用户可以直接随交易发送（金额可以在 `msg.value` 中查到）。

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

为了将以太币存入合约，首先将其包装成包装以太币，然后将包装以太币转入配对。 请注意 转账将打包进 `assert` 中。 这意味着如果转账失败，此合约调用也会失败， 因此包装不会真的发生。

```solidity
        liquidity = IUniswapV2Pair(pair).mint(to);
        // refund dust eth, if any
        if (msg.value > amountETH) TransferHelper.safeTransferETH(msg.sender, msg.value - amountETH);
    }
```

用户已经向我们发送了以太币，如果还有任何额外的资金剩余（因为其他代币 比用户认定的价值更低），我们需要签发退款。

**撤回流动资金**

下面的函数将撤回流动资金并还给流动资金提供者。

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

最简单的流动资金撤回案例。 流动资金提供者同意 接受每种代币有一个最低数额，必须在截止时间之前完成。

```solidity
        address pair = UniswapV2Library.pairFor(factory, tokenA, tokenB);
        IUniswapV2Pair(pair).transferFrom(msg.sender, pair, liquidity); // send liquidity to pair
        (uint amount0, uint amount1) = IUniswapV2Pair(pair).burn(to);
```

核心合约的 `burn` 函数处理返还给用户的代币。

```solidity
        (address token0,) = UniswapV2Library.sortTokens(tokenA, tokenB);
```

某个函数返回多个值时，如果我们只对其中几个值感兴趣，以下便是 我们只获取那些值的方式。 从消耗燃料的角度来说，这样比读取那些从来不用的值更加经济。

```solidity
        (amountA, amountB) = tokenA == token0 ? (amount0, amount1) : (amount1, amount0);
```

将按从核心合约返回代币的路径（按代币地址降序）调整为 以用户期望的方式（对应于 `tokenA` 和 `tokenB`）。

```solidity
        require(amountA >= amountAMin, 'UniswapV2Router: INSUFFICIENT_A_AMOUNT');
        require(amountB >= amountBMin, 'UniswapV2Router: INSUFFICIENT_B_AMOUNT');
    }
```

可以首先进行代币转让，然后再核实转让是否合法，因为如果不合法，我们可以恢复 所有的状态更改。

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

撤回以太币流动资金的方式几乎是一样的，区别在于我们首先会收到包装以太币代币，然后将它们兑换为 以太币，最后再退还给流动资金提供者。

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

这些函数转发元交易，通过[许可证机制](uniswapv2-contracts-guide.md#UniswapV2ERC20)使没有以太币的用户能够从流动池中提取资金。

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

此函数可以用于在传输或存储时收取费用的代币。 当代币合约中有这种费用时，我们不能依靠 `removeLiquidity` 函数来告诉我们可以撤回多少代币。因此，我们需要先撤回然后查询代币金额。

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

最后这个函数将存储费用计入元交易。

**交易**

```solidity
    // **** SWAP ****
    // requires the initial amount to have already been sent to the first pair
    function _swap(uint[] memory amounts, address[] memory path, address _to) internal virtual {
```

呈现给交易者的函数可以调用此函数 以执行内部处理。

```solidity
        for (uint i; i < path.length - 1; i++) {
```

在撰写此教程时，已有 [388,160 个 ERC-20 代币](https://etherscan.io/tokens)。 如果每个代币对 都有配对交易，配对交易数将超过 1500 亿次。 目前为止， 整个链上[只拥有该帐户数量的 0.1%](https://etherscan.io/chart/address)。 实际上，兑换 函数支持路径概念。 交易者可以将 A 兑换成 B、B 兑换成 C、C 兑换成 D，因此 不需要直接的 A-D 配对交易。

这些市场上的价格往往是同步的，因为当它们没有同步时， 就会为套利创造机会。 设想一下，例如有三种代币，A、B 和 C。有三次配对交易， 每一对一次。

1. 初始情况
2. 交易者出售 24.695 A 代币，获得 25.305 B 代币。
3. 交易者卖出了 24.695 B 代币以得到 25.305 C 代币，大约获得 0.61 B 代币的利润。
4. 交易者卖出了 24.695 C 代币以得到 25.305 A 代币，大约获得 0.61 C 代币的利润。 交易者还拥有剩下的 0.61 A 代币（交易者最终拥有的 25.305 A 代币，减去原始投资 24.695 A 代币）。

| 步骤 | A-B 兑换                      | B-C 兑换                      | A-C 兑换                      |
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

获取我们当前处理的配对，排序后（以便与配对一起使用）获得预期的输出金额。

```solidity
            (uint amount0Out, uint amount1Out) = input == token0 ? (uint(0), amountOut) : (amountOut, uint(0));
```

获得预期的金额后，按配对交易所需方式排序。

```solidity
            address to = i < path.length - 2 ? UniswapV2Library.pairFor(factory, output, path[i + 2]) : _to;
```

这是最后一次兑换吗？ 如果是，将收到用于交易的代币发送到目的地址。 如果不是，则将代币发送到 下一个配对交易。

```solidity
            IUniswapV2Pair(UniswapV2Library.pairFor(factory, input, output)).swap(
                amount0Out, amount1Out, to, new bytes(0)
            );
        }
    }
```

真正调用配对交易来兑换代币。 我们不需要回调函数来了解交易信息， 因此没有在该字段中发送任何字节。

```solidity
    function swapExactTokensForTokens(
```

交易者直接使用此函数来兑换代币。

```solidity
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
```

此参数包含 ERC-20 合约的地址。 如上文所述，此参数是一个数组，因为可能 需要通过多次配对交易来从现有资产获取到想要的资产。

Solidity 中的函数参数可以存入 `memory` 或者 `calldata`。 如果此函数是合约的一个入口点， 即直接由用户（通过交易）或从另一个合约调用，那么参数的值 可以直接从调用数据中获取。 如果函数是通过内部调用，如上述 `_swap`，则参数 必须存储在 `memory` 中。 从所调用合约的角度来看，`calldata` 为只读变量。

对于标量类型，如 `uint` 或者 `address`，编译器可以帮助我们处理存储方式的选择，但对于数组， 由于它们需要更多的存储空间也消耗更多的燃料，我们需要指定要使用的存储类型。

```solidity
        address to,
        uint deadline
    ) external virtual override ensure(deadline) returns (uint[] memory amounts) {
```

返回值总是返回内存中。

```solidity
        amounts = UniswapV2Library.getAmountsOut(factory, amountIn, path);
        require(amounts[amounts.length - 1] >= amountOutMin, 'UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT');
```

计算每次兑换时要购买的代币金额。 如果结果低于交易者愿意接受的最低限度， 则撤销交易。

```solidity
        TransferHelper.safeTransferFrom(
            path[0], msg.sender, UniswapV2Library.pairFor(factory, path[0], path[1]), amounts[0]
        );
        _swap(amounts, path, to);
    }
```

最后，将初始的 ERC-20 代币转到第一个配对交易的帐户中，然后调用 `_swap`。 所有这些 都发生在同一次交易中，因此配对交易知道任何意料之外的代币都是此次转账的一部分。

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

前一个函数，`swapTokensForTokens`，使交易者可以指定愿意 给出代币的准确数量和愿意接受代币的最低数量。 此函数可以撤销兑换， 使交易者能够指定想要的输出代币数额，以及愿意支付的输入代币的最大数额。

在这两种情况下，交易者必须首先给予此外围合约一定的额度，用于转账。

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

这四种转换方式都涉及到以太币和代币之间的交易。 唯一不同的是，我们要么从交易者处收到以太币， 并使用以太币铸造包装以太币，或者从路径上最后一次交易收到包装以太币， 消耗后将得到的以太币再发送给交易者。

```solidity
    // **** SWAP (supporting fee-on-transfer tokens) ****
    // requires the initial amount to have already been sent to the first pair
    function _swapSupportingFeeOnTransferTokens(address[] memory path, address _to) internal virtual {
```

此内部函数用于兑换代币，但有转账或存储费用，以解决 （[此问题](https://github.com/Uniswap/uniswap-interface/issues/835)）。

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

由于有转账费用，我们不能依靠 `getAmountsOut` 函数来告诉我们 每次转账完成后获得的金额（之前我们可以调用原来的 `_swap`）。 相反，我们必须先完成转账然后再查看 我们收回的代币数量。

注意：理论上我们可以使用此函数而非 `_swap`，但在某些情况下（例如， 如果因为在最后无法满足所需最低数额而导致转账撤销），最终会消耗更多 燃料。 转账需要收费的代币很少见，所以，尽管我们需要接纳它们，但不需要让所有的兑换都假定 至少需要兑换一种需要收取转账费用的代币。

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

这些方式与用于普通代币的相同，区别在于它们调用的是`_swapSupportingFeeOnTransferTokens`。

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

这些函数仅仅是调用 [UniswapV2Library 函数](uniswapv2-contracts-guide.md#uniswapV2library)的代理。

#### UniswapV2Migrator.sol <a href="#uniswapv2migrator" id="uniswapv2migrator"></a>

这个合约用于将交易从旧版 v1 迁移至 v2。 目前版本已经迁移，便不再相关。

### 程序库 <a href="#libraries" id="libraries"></a>

[SafeMath 库](https://docs.openzeppelin.com/contracts/2.x/api/math)是一个文档很完备的程序库，这里 便无需赘述了。

#### 数学 <a href="#math" id="math"></a>

此库包含一些 Solidity 代码通常不需要的数学函数，因而它们不是 Solidity 语言的一部分。

```solidity
pragma solidity =0.5.16;

// a library for performing various math operations

library Math {
    function min(uint x, uint y) internal pure returns (uint z) {
        z = x < y ? x : y;
    }

    // babylonian method (https://wikipedia.org/wiki/Methods_of_computing_square_roots#Babylonian_method)
    function sqrt(uint y) internal pure returns (uint z) {
        if (y > 3) {
            z = y;
            uint x = y / 2 + 1;
```

首先赋予 x 一个大于平方根的估值（这是我们需要把 1-3 当作特殊情况处理的原因）。

```solidity
            while (x < z) {
                z = x;
                x = (y / x + x) / 2;
```

获取一个更接近的估值，即前一个估值与我们试图找到的方根值的平均数除以 前一个估值。 重复计算，直到新的估值不再低于现有估值。 欲了解更多详情， [请参见此处](https://wikipedia.org/wiki/Methods\_of\_computing\_square\_roots#Babylonian\_method)。

```solidity
            }
        } else if (y != 0) {
            z = 1;
```

我们永远不需要零的平方根。 1、2 和 3 的平方根大致为 1（我们使用的是 整数，所以忽略分数）。

```solidity
        }
    }
}
```

#### 定点小数 (UQ112x112) <a href="#fixedpoint" id="fixedpoint"></a>

该库处理小数，这些小数通常不属于以太坊计算的一部分。 为此，它将数值 _x_ 编码为 _x\*2^112_。 这使我们能够使用原来的加法和减法操作码，无需更改。

```solidity
pragma solidity =0.5.16;

// a library for handling binary fixed point numbers (https://wikipedia.org/wiki/Q_(number_format))

// range: [0, 2**112 - 1]
// resolution: 1 / 2**112

library UQ112x112 {
    uint224 constant Q112 = 2**112;
```

`Q112` 是 1 的编码。

```solidity
    // encode a uint112 as a UQ112x112
    function encode(uint112 y) internal pure returns (uint224 z) {
        z = uint224(y) * Q112; // never overflows
    }
```

因为 y 是`uint112`，所以最多可以是 2^112-1。 该数值还可以编码为 `UQ112x112`。

```solidity
    // divide a UQ112x112 by a uint112, returning a UQ112x112
    function uqdiv(uint224 x, uint112 y) internal pure returns (uint224 z) {
        z = x / uint224(y);
    }
}
```

如果我们需要两个 `UQ112x112` 值相除，结果不需要再乘以 2^112。 因此， 我们为分母取一个整数。 我们需要使用类似的技巧来做乘法，但不需要将 `UQ112x112` 的值相乘。

#### UniswapV2Library <a href="#uniswapv2library" id="uniswapv2library"></a>

此库仅被外围合约使用

```solidity
pragma solidity >=0.5.0;

import '@uniswap/v2-core/contracts/interfaces/IUniswapV2Pair.sol';

import "./SafeMath.sol";

library UniswapV2Library {
    using SafeMath for uint;

    // returns sorted token addresses, used to handle return values from pairs sorted in this order
    function sortTokens(address tokenA, address tokenB) internal pure returns (address token0, address token1) {
        require(tokenA != tokenB, 'UniswapV2Library: IDENTICAL_ADDRESSES');
        (token0, token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
        require(token0 != address(0), 'UniswapV2Library: ZERO_ADDRESS');
    }
```

按地址对这两个代币排序，所以我们将能够获得相应的配对交易地址。 这很有必要， 否则有两种可能性，一种是用于参数 A,B，而另一种是用于 参数 B,A，导致两次交易而非一个。

```solidity
    // calculates the CREATE2 address for a pair without making any external calls
    function pairFor(address factory, address tokenA, address tokenB) internal pure returns (address pair) {
        (address token0, address token1) = sortTokens(tokenA, tokenB);
        pair = address(uint(keccak256(abi.encodePacked(
                hex'ff',
                factory,
                keccak256(abi.encodePacked(token0, token1)),
                hex'96e8ac4277198ff8b6f785478aa9a39f403cb768dd02cbee326c3e7da348845f' // init code hash
            ))));
    }
```

此函数计算两种代币的配对交易地址。 此合约使用 [CREATE2 操作码](https://eips.ethereum.org/EIPS/eip-1014)创建，如果我们知道所使用的参数， 我们可以使用相同的算法计算地址。 这比查询工厂便宜得多，而且

```solidity
    // fetches and sorts the reserves for a pair
    function getReserves(address factory, address tokenA, address tokenB) internal view returns (uint reserveA, uint reserveB) {
        (address token0,) = sortTokens(tokenA, tokenB);
        (uint reserve0, uint reserve1,) = IUniswapV2Pair(pairFor(factory, tokenA, tokenB)).getReserves();
        (reserveA, reserveB) = tokenA == token0 ? (reserve0, reserve1) : (reserve1, reserve0);
    }
```

此函数返回配对交易所拥有的两种代币的储备金。 请注意，它可以任意顺序接收代币， 并将其排序，以便内部使用。

```solidity
    // given some amount of an asset and pair reserves, returns an equivalent amount of the other asset
    function quote(uint amountA, uint reserveA, uint reserveB) internal pure returns (uint amountB) {
        require(amountA > 0, 'UniswapV2Library: INSUFFICIENT_AMOUNT');
        require(reserveA > 0 && reserveB > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');
        amountB = amountA.mul(reserveB) / reserveA;
    }
```

如果不涉及交易费用的话，此函数将返回给您代币 A 兑换得到的代币 B。 此计算 考虑到转账可能会改变汇率。

```solidity
    // given an input amount of an asset and pair reserves, returns the maximum output amount of the other asset
    function getAmountOut(uint amountIn, uint reserveIn, uint reserveOut) internal pure returns (uint amountOut) {
```

如果使用配对交易没有手续费，上述 `quote` 函数非常有效。 然而，如果有 0.3% 的 手续费，您实际得到的金额就会低于此值。 此函数可以计算缴纳交易费用后的金额。

```solidity
        require(amountIn > 0, 'UniswapV2Library: INSUFFICIENT_INPUT_AMOUNT');
        require(reserveIn > 0 && reserveOut > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');
        uint amountInWithFee = amountIn.mul(997);
        uint numerator = amountInWithFee.mul(reserveOut);
        uint denominator = reserveIn.mul(1000).add(amountInWithFee);
        amountOut = numerator / denominator;
    }
```

Solidity 本身不能进行小数计算，所以不能简单地将金额乘以 0.997。 作为替代方法， 我们将分子乘以 997，分母乘以 1000，也能取得相同的效果。

```solidity
    // given an output amount of an asset and pair reserves, returns a required input amount of the other asset
    function getAmountIn(uint amountOut, uint reserveIn, uint reserveOut) internal pure returns (uint amountIn) {
        require(amountOut > 0, 'UniswapV2Library: INSUFFICIENT_OUTPUT_AMOUNT');
        require(reserveIn > 0 && reserveOut > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');
        uint numerator = reserveIn.mul(amountOut).mul(1000);
        uint denominator = reserveOut.sub(amountOut).mul(997);
        amountIn = (numerator / denominator).add(1);
    }
```

此函数大致完成相同的功能，但它会获取输出数额并提供输入代币的数量。

```solidity
    // performs chained getAmountOut calculations on any number of pairs
    function getAmountsOut(address factory, uint amountIn, address[] memory path) internal view returns (uint[] memory amounts) {
        require(path.length >= 2, 'UniswapV2Library: INVALID_PATH');
        amounts = new uint[](path.length);
        amounts[0] = amountIn;
        for (uint i; i < path.length - 1; i++) {
            (uint reserveIn, uint reserveOut) = getReserves(factory, path[i], path[i + 1]);
            amounts[i + 1] = getAmountOut(amounts[i], reserveIn, reserveOut);
        }
    }

    // performs chained getAmountIn calculations on any number of pairs
    function getAmountsIn(address factory, uint amountOut, address[] memory path) internal view returns (uint[] memory amounts) {
        require(path.length >= 2, 'UniswapV2Library: INVALID_PATH');
        amounts = new uint[](path.length);
        amounts[amounts.length - 1] = amountOut;
        for (uint i = path.length - 1; i > 0; i--) {
            (uint reserveIn, uint reserveOut) = getReserves(factory, path[i - 1], path[i]);
            amounts[i - 1] = getAmountIn(amounts[i], reserveIn, reserveOut);
        }
    }
}
```

在需要进行数次配对交易时，可以通过这两个函数获得相应数值。

#### 转账帮助 <a href="#transfer-helper" id="transfer-helper"></a>

[此库](https://github.com/Uniswap/uniswap-lib/blob/master/contracts/libraries/TransferHelper.sol)添加了围绕 ERC-20 和以太坊转账的成功检查，并以同样的方式处理回退和返回 `false` 值。

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later

pragma solidity >=0.6.0;

// helper methods for interacting with ERC20 tokens and sending ETH that do not consistently return true/false
library TransferHelper {
    function safeApprove(
        address token,
        address to,
        uint256 value
    ) internal {
        // bytes4(keccak256(bytes('approve(address,uint256)')));
        (bool success, bytes memory data) = token.call(abi.encodeWithSelector(0x095ea7b3, to, value));
```

我们可以通过以下两种方式调用不同的合约：

* 使用一个接口定义创建函数调用
* 使用 [应用程序二进制接口 (ABI)](https://docs.soliditylang.org/en/v0.8.3/abi-spec.html)“手动” 创建调用。 这是代码作者的决定。

```solidity
        require(
            success && (data.length == 0 || abi.decode(data, (bool))),
            'TransferHelper::safeApprove: approve failed'
        );
    }
```

为了与之前的 ERC-20 标准创建的代币反向兼容，ERC-20 调用 失败可能有两种情况：回退（在这种情况下 `success` 即是 `false`），或者调用成功但返回 `false` 值（在这种情况下有输出数据，将其解码为布尔值，会得到 `false`）。

```solidity
    function safeTransfer(
        address token,
        address to,
        uint256 value
    ) internal {
        // bytes4(keccak256(bytes('transfer(address,uint256)')));
        (bool success, bytes memory data) = token.call(abi.encodeWithSelector(0xa9059cbb, to, value));
        require(
            success && (data.length == 0 || abi.decode(data, (bool))),
            'TransferHelper::safeTransfer: transfer failed'
        );
    }
```

此函数实现了 [ERC-20 的转账功能](https://eips.ethereum.org/EIPS/eip-20#transfer)， 可使一个帐户花掉由不同帐户所提供的额度。

```solidity
    function safeTransferFrom(
        address token,
        address from,
        address to,
        uint256 value
    ) internal {
        // bytes4(keccak256(bytes('transferFrom(address,address,uint256)')));
        (bool success, bytes memory data) = token.call(abi.encodeWithSelector(0x23b872dd, from, to, value));
        require(
            success && (data.length == 0 || abi.decode(data, (bool))),
            'TransferHelper::transferFrom: transferFrom failed'
        );
    }
```

此函数实现了 [ERC-20 的 transferFrom 功能](https://eips.ethereum.org/EIPS/eip-20#transferfrom)， 可使一个帐户花掉由不同帐户所提供的额度。

```solidity
    function safeTransferETH(address to, uint256 value) internal {
        (bool success, ) = to.call{value: value}(new bytes(0));
        require(success, 'TransferHelper::safeTransferETH: ETH transfer failed');
    }
}
```

此函数将以太币转至一个帐户。 任何对不同合约的调用都可以尝试发送以太币。 因为我们 实际上不需要调用任何函数，就不需要在调用中发送数据。

### 结论 <a href="#conclusion" id="conclusion"></a>

本篇文章较长，约有 50 页。 如果您已读到此处，恭喜您！ 希望您现在已经了解 编写真实应用程序（相对于短小的示例程序）的考虑因素，并且能够更好地为您自己的 用例编写合约。

现在去写点实用的东西吧，希望您能给我们惊喜。
