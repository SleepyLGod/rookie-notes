# Uniswap V2 Pair

### 核心合约与外围合约 <a href="#contract-types" id="contract-types"></a>

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

1. 先乐观地将输出代币发送到目的地址；如果 `data` 非空，还会触发 flash swap 回调。
2. 读取 Pair 当前余额，并根据余额和旧 reserve 的差值推导本次实际输入了多少代币。
3. 用扣除手续费后的余额校验恒定乘积 invariant，确保核心合约没有被欺骗。
4. 调用 `_update` 更新 reserve。

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

[SafeMath 库](https://docs.openzeppelin.com/contracts/3.x/api/math)用于避免整数上溢和 下溢。 这很重要，否则最终可能会出现这样的情况：本该是 `-1` 的值， 结果却成了 `2^256-1`。

```solidity
    using UQ112x112 for uint224;
```

流动池合约中的许多计算都需要分数。 但是，以太坊虚拟机本身不支持分数。 Uniswap 找到的解决方案是使用 224 位数值，整数值为 112 位，分数部分 为 112 位。 因此，`1.0` 用 `2^112` 表示，`1.5` 用 `2^112 + 2^111` 表示，以此类推。

关于这个函数库的更详细内容在[文档的稍后部分](README.md#FixedPoint)。

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

如果有新的流动性变化需要收取协议费。 你可以在 [本文后面](README.md#Math)看到平方根函数。

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

当流动资金提供者为资金池增加流动资金时，将会调用此函数。 它将产生额外的流动池 代币作为奖励。 在 [外围合约](README.md#UniswapV2Router02)中增加流动性后调用这个函数，以确保二者在同一交易中（因此其他人都不能提交向合法所有者要求新流动资金的交易）。

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

当流动资金被提取且相应的流动池代币需要被销毁时，将调用此函数。 还需要[从一个外围账户](README.md#UniswapV2Router02)调用。

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

此函数也应该从[外围合约](README.md#UniswapV2Router02)调用。

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
