# Uniswap V2

## 😡 Uniswap-v2 合约概览

### 介绍 <a href="#introduction" id="introduction"></a>

[Uniswap v2](https://uniswap.org/whitepaper.pdf) 可以在任何两个 ERC-20 代币之间创建一个兑换市场。 在这篇文章中， 我们将了解实现此协议的合约的源代码，看看为什么要 这样写代码。

#### Uniswap 是做什么的？ <a href="#what-does-uniswap-do" id="what-does-uniswap-do"></a>

一般来说有两类用户：流动资金提供者和交易者。

_流动资金提供者\_为资金池提供两种可以兑换的代币（我们称之为 **Token0** 和 **Token1**）。 作为回报，他们会收到第三种代币，代表对资金池的 部分所有权，这个池叫做\_流动代币_。

\_交易者\_将一种代币发送到资金池，并从流动资金提供者的资金池中接收另一种代币（例如，发送 **Token0** 并获得 **Token1**）。 兑换汇率由 **Token0** 和 **Token1** 的相对数量决定。 此外，资金池将收取汇率的一小部分作为流动资金池的奖励。

当流动资金提供者想要收回他们的代币资产时，他们可以消耗资金池代币并收回他们的代币， 其中包括他们在兑换过程中奖励的份额。

[点击这里查看更完整的描述](https://uniswap.org/docs/v2/core-concepts/swaps/)。

#### 为什么选择 v2？ 而不是 v3？ <a href="#why-v2" id="why-v2"></a>

这篇笔记以 Uniswap v2 为学习对象，因为 v2 的恒定乘积 AMM、Pair/Factory/Router 分层和 LP token 机制更适合作为入门材料。需要注意时间语境：原教程写作时 v3 还接近发布；现在 Uniswap 已经有 v3/v4。实际集成或研究时，应按目标版本重新阅读官方文档和合约。

## Contract notes

- [Pair](pair.md)
- [Factory and ERC20](factory-and-erc20.md)
- [Router](router.md)
- [Libraries](libraries.md)
- [Swap with Router](how-to-swap.md)
