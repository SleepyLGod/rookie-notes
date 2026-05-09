---
coverY: 0
---

# Blockchain

This section mainly contains notes on Uniswap V2 contract reading, mainnet-fork testing, and BigNumber usage.

Suggested reading order:

1. [Uniswap V2 Contract Overview](uniswap-v2/README.md): first understand core/periphery, Pair, Factory, Router, and common flows.
2. [Integrating Uniswap V2 Token Swaps](uniswap-v2/how-to-swap.md): then read the Hardhat mainnet-fork test and Router call flow.
3. [Common arithmetic operations in bignumber.js](bignumber.md): finally review precision handling for on-chain amount calculations.

Content status: Uniswap V2 is still useful as introductory AMM material, but the current ecosystem already has V3 and V4. For real integration, re-check the official documentation and contract addresses for the target protocol version.
