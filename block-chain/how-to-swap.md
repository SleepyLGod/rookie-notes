---
description: 参考原文：https://medium.com/uv-labs/uniswap-testing-1d88ca523bf0
---

# 😭 对接 Uniswap V2 兑换代币

> [**Uniswap**](https://learnblockchain.cn/tags/Uniswap)

对接 Uniswap V2 兑换代币，并测试验证。

![Uniswap & eth](https://img.learnblockchain.cn/pics/20220608144145.jpeg!/scale/60)

在本文中，我们将和正式的 Uniswap V2 交互，实现使用[Uniswap](https://uniswap.org/)进行代币兑换（swap）并通过测试验证兑换功能；

通过测试验证智能合约的行为是一个很好的方式，测试让你相信代码以我们想要的方式执行，而不是以它不应该的方式执行。

在本文中，我们还将学习到如何 fork 主网，并冒充（模拟）一个链上账号进行交易，并编写测试。

## 关于Uniswap V2

但在深入研究之前，为了本文完整，让我们再次介绍一下 Uniswap，Uniswap是一个去中心化的交易所（DEX），运行在以太坊区块链上（主网和其他一些网络）。顾名思义，Uniswap是用来交易ERC20代币的。

Uniswap有3个主要功能:

* 在不同的代币之间进行兑换
* 添加代币对流动性，获得LP ERC-20流动性代币
* 销毁 LP ERC-20流动性代币，取回配对的ERC-20代币

在这篇文章中，我们将重点讨论使用fork 主网在不同的代币之间进行兑换。

**所以让我们开始吧！** 🥳🥳🥳

## 创建一个项目并初始化

在命令行（CLI）上使用以下命令来初始化项目。

```bash
mkdir uni_swap && cd uni_swap
npm init -y
```

安装项目所需的依赖项，运行：

```bash
npm install --save hardhat @nomiclabs/hardhat-ethers @nomiclabs/hardhat-waffle ethers @uniswap/v2-core dotenv
```

## 初始化Hardhat项目

要初始化你的Hardhat项目，在CLI中运行`npx hardhat`命令，并创建一个空的_config.js_文件。

并定制你的Hardhat配置，因为我们要fork主网来与Uniswap交互。因此，Hardhat配置应该看起来类似于这样：

![img](https://img.learnblockchain.cn/pics/20220608144228.png)

注意：用你的自己[Alchemy](https://alchemy.com/?r=7d60e34c-b30a-4ffa-89d4-3c4efea4e14b)API密钥替换URL中的`<key>`部分。

## 编写合约实现兑换

为合约、脚本和测试创建目录，以便更好地组织代码。

在你的CLI中使用以下代码创建目录：

```bash
mkdir contracts && mkdir scripts && mkdir tests
```

为了编写兑换合约，在合约目录内创建一个文件，命名为`testSwap.sol`。

在你的 `testSwap.sol` 中导入Uniswap 等接口，并创建一个名为**testSwap**的合约。

它应该看起来像这样：

![img](https://img.learnblockchain.cn/pics/20220608144250.png)

现在，在`testSwap`中，我们需要包括**Uniswap Router**的地址，我们使用它来完成代币兑换。

使用下面的代码：

```solidity
//address of the uniswap v2 router
address private constant UNISWAP_V2_ROUTER = 0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D;
```

现在，定义要用来兑换的函数：

```solidity
// 兑换函数
    function swap (
        address _tokenIn,
        address _tokenOut,
        uint256 _amountIn,
        address _to,
        uint256 _deadline
    ) external {}
```

函数命名为\*\*swap，\*\*里面有

* **\_tokenIn**： 是我们要兑换的代币的地址。
* **\_tokenOut**：是我们想从这次交易中获得的代币的地址。
* **\_amountIn**： 是我们要交易的代币的数量。
* **\_to**：交易兑换出的代币发送到这个地址。
* **\_deadline**：是交易应该被执行的时间期限。如果超过了最后期限，交易就会失败。

在兑换函数里面，我们要做的第一件事是在合约里面把所需数量的_**\_tokenIn**_ 转移到合约里，使用`msg.sender`：

```solidity
// 把 token 从用户转移到合约
IERC20(_tokenIn).transferFrom(msg.sender, address(this), _amountIn);
```

一旦调用执行，**\_amountIn** 数量的 **\_tokenIn**就会转入到`testSwap`合约中

接下来，通过调用**IERC20** 授权，允许Uniswap合约花费`testSwap`合约中**\_amountIn**数量的代币。

```solidity
// by calling IERC20 approve you allow the uniswap contract to spend the tokens in this contract
IERC20(_tokenIn).approve(UNISWAP_V2_ROUTER, _amountIn);
```

在使用 Uniswap Router 兑换，需要为兑换代币的设置**路径**，路径上第一“站”是使用的代币，最后一“站”期望收到的代币。

所以，我们将声明一个名为`path`的地址数组，填入 **\_tokenIn** 的地址和 **\_tokenOut** 的地址。

```solidity
address[] memory path;
path = new address[](2);
path[0] = _tokenIn; // DAI
path[1] = _tokenOut; // WETH
```

接下来，我们调用函数**getAmountsOut**，以预估可以兑换代币数量，对真实兑换之前预知可兑换数量是很有用的。**getAmountsOut**函数需要一个输入金额和一个代币地址的路径数组：

```solidity
uint256[] memory amountsExpected = IUniswapV2Router(UNISWAP_V2_ROUTER).getAmountsOut(
            _amountIn,
            path
);
```

最后，我们调用 Uniswap Router 的函数 **swapExactTokensForTokens**，并传入参数。这个函数的第一个参数是精确输入金额 `amountIn`，第二个参数才是最小可接受输出金额 `amountOutMin`；不要把预估输出数组里的值误当成输入语义。

```solidity
uint256[] memory amountsReceived = IUniswapV2Router(UNISWAP_V2_ROUTER).swapExactTokensForTokens(
            _amountIn,
            (amountsExpected[1]*990)/1000, // 接受 1% 的滑点
            path,
            _to,
            _deadline
);
```

**恭喜你**! 我们的的兑换合约已经准备好了。🎉

完整的看起来应该类似是这样：

![img](https://img.learnblockchain.cn/pics/20220608144306.png)

使用命令`npx hardhat compile`来检查我们的智能合约中是否有错误。

现在，是时候为我们的合约运行一些测试了

## 编写测试脚本

在_tests_文件夹中创建一个文件，并将其命名为\*\*\*`sample-test.js`\*\*\*。

首先，要从Uniswap导入ERC20合约的ABI，同时，定义测试的结构和我们要使用的合约的地址。

```javascript
const ERC20ABI = require("@uniswap/v2-core/build/ERC20.json").abi;

describe("Test Swap", function () {
    const DAIAddress = "0x6B175474E89094C44Da98b954EedeAC495271d0F";
    const WETHAddress = "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2";
    const MyAddress = "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B";
    const DAIHolder = "0x5d38b4e4783e34e2301a2a36c39a03c45798c4dd";
}
```

这里，我们使用了4个地址：

* **DAIAddress**和**WETHAddress**分别是Dai 合约和WETH 合约的地址，它们将在交易中使用
* **MyAddress**是交易者的地址。
* **DAIHolder**是我们要冒充的地址。

现在，在编写测试脚本之前，我们将部署**testSwap**智能合约。为此，我们使用以下代码：

```javascript
let TestSwapContract;

beforeEach(async () => {
        const TestSwapFactory = await ethers.getContractFactory("testSwap");
        TestSwapContract = await TestSwapFactory.deploy();
        await TestSwapContract.deployed();
})

beforeEach(async () => {
        const TestSwapFactory = await ethers.getContractFactory("testSwap");
        TestSwapContract = await TestSwapFactory.deploy();
        await TestSwapContract.deployed();
})
```

为测试脚本创建一个测试用例，并“冒充”我们之前定义的**DAIHolder**地址。

```javascript
it("should swap", async () => { 
            await hre.network.provider.request({
            method: "hardhat_impersonateAccount",
            params: [DAIHolder],
});
const impersonateSigner = await ethers.getSigner(DAIHolder);
```

在下一步，我们将通过使用冒充的账户获得其**DAI代币**的初始余额。之后，我们将使用该余额进行兑换交易。

同样，我们也获取**WETH代币**的余额，以便观察代币的兑换情况。

```javascript
const DAIContract = new ethers.Contract(DAIAddress, ERC20ABI, impersonateSigner)
const DAIHolderBalance = await DAIContract.balanceOf(impersonateSigner.address)

const WETHContract = new ethers.Contract(WETHAddress, ERC20ABI, impersonateSigner)

const myBalance = await WETHContract.balanceOf(MyAddress);
console.log("Initial WETH Balance:", ethers.utils.formatUnits(myBalance.toString()));
```

然后，我们将使用DAI合约来批准（授权）TestSwap 可使用兑换的金额：

```javascript
await DAIContract.approve(TestSwapContract.address, DAIHolderBalance)
```

对于最后兑换截止时间，先获取最新区块的当前时间戳：

```javascript
// getting current timestamp
const latestBlock = await ethers.provider.getBlockNumber();
const timestamp = (await ethers.provider.getBlock(latestBlock)).timestamp;
```

通过调用我们编写的**swap**函数进行交易。传入我们在上面配置的参数：

这个交易将从通过**DAIHolder**发起：

```javascript
await TestSwapContract.connect(impersonateSigner).swap(
            DAIAddress,
            WETHAddress,
            DAIHolderBalance,
            MyAddress,
            timestamp + 1000 // adding 100 milliseconds to the current blocktime
)
```

最后，验证兑换交易：

```javascript
const myBalance_updated = await WETHContract.balanceOf(MyAddress);
console.log("Balance after Swap:", ethers.utils.formatUnits(myBalance_updated.toString()));
const DAIHolderBalance_updated = await DAIContract.balanceOf(impersonateSigner.address);
```

在这里，检查了兑换功能执行后我们账户的余额。

在这下面，我们写了一些测试以检查交易是否真实完成：

```javascript
expect(DAIHolderBalance_updated.eq(BigNumber.from(0))).to.be.true
expect(myBalance_updated.gt(myBalance)).to.be.true;
```

* 由于我们使用了所有的余额进行交易，因此在第一个测试中，我们期望DAI代币余额应该等于0。
* 在第二个测试中，检查我们账户中的**余额**是否比之前的大。

因此，这就是我们要进行的两个测试。

sample-test.js 应该类似于下面的样子，请注意文件开头的 `require` 语句：

![img](https://img.learnblockchain.cn/pics/20220608144317.png)

当然，请自由探索，用它们尝试更多的测试。

现在，我们要用`npx hardhat test`命令来运行这些测试。

结果应该是这样的：

![img](https://img.learnblockchain.cn/pics/20220608144853.png)

正如你所看到的，我们的初始余额在兑换完成后有所增加。

而我们编写的测试也成功了！！。🎉🎉🎉

如果你一直跟到最后，那么恭喜你，你已经做得很好了。
