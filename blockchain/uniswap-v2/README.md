# Uniswap V2

## Uniswap V2 Contract Overview

### Introduction <a href="#introduction" id="introduction"></a>

[Uniswap v2](https://uniswap.org/whitepaper.pdf) can create an exchange market between any two ERC20 tokens. In this note, we look at the source code of the contracts that implement this protocol and examine why the code is written this way.

#### What does Uniswap do? <a href="#what-does-uniswap-do" id="what-does-uniswap-do"></a>

In general, there are two types of users: liquidity providers and traders.

_Liquidity providers_ provide two tokens that can be exchanged in the pool, which we call **Token0** and **Token1**. In return, they receive a third token that represents partial ownership of the pool. This third token is the _liquidity token_.

_Traders_ send one token into the pool and receive another token from the liquidity providers' pool. For example, they may send **Token0** and receive **Token1**. The exchange rate is determined by the relative quantities of **Token0** and **Token1**. In addition, the pool takes a small portion of the exchange rate as a reward for the liquidity pool.

When liquidity providers want to reclaim their token assets, they can burn their pool tokens and withdraw their underlying tokens, including their share of the rewards earned through swaps.

[Click here for a more complete description](https://uniswap.org/docs/v2/core-concepts/swaps/).

#### Why v2 instead of v3? <a href="#why-v2" id="why-v2"></a>

This note uses Uniswap V2 as the learning target because V2's constant-product AMM, Pair/Factory/Router layering, and LP token mechanism are more suitable as introductory material. The time context matters: when the original tutorial was written, V3 was still close to release; now Uniswap already has V3 and V4. For real integration or research, reread the official documentation and contracts for the target version.

## Contract notes

- [Pair](pair.md)
- [Factory and ERC20](factory-and-erc20.md)
- [Router](router.md)
- [Libraries](libraries.md)
- [Swap with Router](how-to-swap.md)
