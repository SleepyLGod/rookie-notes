# Uniswap V2 Router

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

> 原教程这里有一张流动性注入流程图，但图片文件没有随当前笔记仓库保存。

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

这些函数转发元交易，通过[许可证机制](README.md#UniswapV2ERC20)使没有以太币的用户能够从流动池中提取资金。

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

这些函数仅仅是调用 [UniswapV2Library 函数](README.md#uniswapV2library)的代理。

#### UniswapV2Migrator.sol <a href="#uniswapv2migrator" id="uniswapv2migrator"></a>

这个合约用于将交易从旧版 v1 迁移至 v2。 目前版本已经迁移，便不再相关。
