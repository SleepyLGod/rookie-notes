---
coverY: 0
---

# Blockchain

这一节主要是 Uniswap V2 合约阅读、主网 fork 测试和 BigNumber 使用笔记。

建议阅读顺序：

1. [Uniswap V2 合约概览](uniswap-v2/contracts-guide.md)：先理解 core/periphery、Pair、Factory、Router 和常见流程。
2. [对接 Uniswap V2 兑换代币](uniswap-v2/how-to-swap.md)：再看 Hardhat fork 主网测试和 Router 调用。
3. [bignumber.js 中常见运算](bignumber.md)：最后补齐链上金额计算的精度处理。

内容状态：Uniswap V2 仍适合作为 AMM 入门材料，但当前生态已经有 v3/v4。涉及实际集成时，需要按目标协议版本重新核对官方文档和合约地址。
