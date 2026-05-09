# Uniswap V2 Libraries

### Libraries <a href="#libraries" id="libraries"></a>

The [SafeMath library](https://docs.openzeppelin.com/contracts/3.x/api/math) is well documented, so this note does not repeat its details.

#### Math <a href="#math" id="math"></a>

This library contains mathematical functions that Solidity code usually does not need, so they are not part of the Solidity language itself.

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

First, assign `x` an estimate greater than the square root. This is why values from 1 to 3 need to be handled as special cases.

```solidity
            while (x < z) {
                z = x;
                x = (y / x + x) / 2;
```

Compute a closer estimate: the average of the previous estimate and the value obtained by dividing the target by that previous estimate. Repeat until the new estimate is no longer lower than the existing estimate. For more details, [see the Babylonian method](https://wikipedia.org/wiki/Methods\_of\_computing\_square\_roots#Babylonian\_method).

```solidity
            }
        } else if (y != 0) {
            z = 1;
```

The square root of zero is never needed in this branch. The square roots of 1, 2, and 3 are approximately 1 in integer arithmetic because fractional parts are ignored.

```solidity
        }
    }
}
```

#### Fixed Point Numbers (UQ112x112) <a href="#fixedpoint" id="fixedpoint"></a>

This library handles fractional values, which are usually not part of Ethereum computation. It encodes a value _x_ as _x\*2^112_. This lets the contract use the original addition and subtraction opcodes without modification.

```solidity
pragma solidity =0.5.16;

// a library for handling binary fixed point numbers (https://wikipedia.org/wiki/Q_(number_format))

// range: [0, 2**112 - 1]
// resolution: 1 / 2**112

library UQ112x112 {
    uint224 constant Q112 = 2**112;
```

`Q112` is the encoding of 1.

```solidity
    // encode a uint112 as a UQ112x112
    function encode(uint112 y) internal pure returns (uint224 z) {
        z = uint224(y) * Q112; // never overflows
    }
```

Because `y` is a `uint112`, its maximum value is `2^112 - 1`. That value can still be encoded as `UQ112x112`.

```solidity
    // divide a UQ112x112 by a uint112, returning a UQ112x112
    function uqdiv(uint224 x, uint112 y) internal pure returns (uint224 z) {
        z = x / uint224(y);
    }
}
```

If two `UQ112x112` values were divided directly, the result would not need to be multiplied by `2^112` again. Therefore, this helper takes an integer denominator. A similar trick would be needed for multiplication, but Uniswap V2 does not need to multiply two `UQ112x112` values here.

#### UniswapV2Library <a href="#uniswapv2library" id="uniswapv2library"></a>

This library is used only by the peripheral contracts.

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

The two token addresses are sorted so the corresponding pair address can be obtained deterministically. This is necessary because otherwise there would be two possibilities: one for parameters `A, B`, and another for parameters `B, A`, resulting in two exchanges instead of one.

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

This function calculates the pair address for two tokens. Since the pair contract is created with the [CREATE2 opcode](https://eips.ethereum.org/EIPS/eip-1014), if we know the parameters used for creation, we can compute the address with the same algorithm. This is much cheaper than querying the factory and does not require an external call.

```solidity
    // fetches and sorts the reserves for a pair
    function getReserves(address factory, address tokenA, address tokenB) internal view returns (uint reserveA, uint reserveB) {
        (address token0,) = sortTokens(tokenA, tokenB);
        (uint reserve0, uint reserve1,) = IUniswapV2Pair(pairFor(factory, tokenA, tokenB)).getReserves();
        (reserveA, reserveB) = tokenA == token0 ? (reserve0, reserve1) : (reserve1, reserve0);
    }
```

This function returns the reserves of the two tokens held by the pair. Notice that it can receive the token addresses in any order, then sorts them for internal use.

```solidity
    // given some amount of an asset and pair reserves, returns an equivalent amount of the other asset
    function quote(uint amountA, uint reserveA, uint reserveB) internal pure returns (uint amountB) {
        require(amountA > 0, 'UniswapV2Library: INSUFFICIENT_AMOUNT');
        require(reserveA > 0 && reserveB > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');
        amountB = amountA.mul(reserveB) / reserveA;
    }
```

If there were no trading fee, this function would return how much token B corresponds to a given amount of token A. The calculation accounts for the reserves because the swap changes the exchange rate.

```solidity
    // given an input amount of an asset and pair reserves, returns the maximum output amount of the other asset
    function getAmountOut(uint amountIn, uint reserveIn, uint reserveOut) internal pure returns (uint amountOut) {
```

The `quote` function above works well when using a pair has no fee. However, with a 0.3% trading fee, the amount actually received is lower than the quoted value. This function calculates the amount after the trading fee is applied.

```solidity
        require(amountIn > 0, 'UniswapV2Library: INSUFFICIENT_INPUT_AMOUNT');
        require(reserveIn > 0 && reserveOut > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');
        uint amountInWithFee = amountIn.mul(997);
        uint numerator = amountInWithFee.mul(reserveOut);
        uint denominator = reserveIn.mul(1000).add(amountInWithFee);
        amountOut = numerator / denominator;
    }
```

Solidity itself does not support fractional arithmetic, so the contract cannot simply multiply the amount by `0.997`. Instead, it multiplies the numerator by `997` and the denominator by `1000`, which produces the same effect in integer arithmetic.

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

This function performs the inverse calculation: it receives the desired output amount and returns the required input token amount.

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

When a route requires multiple pair swaps, these two functions compute the corresponding amounts across the full path.

#### Transfer Helper <a href="#transfer-helper" id="transfer-helper"></a>

[This library](https://github.com/Uniswap/uniswap-lib/blob/master/contracts/libraries/TransferHelper.sol) adds success checks around ERC20 and ETH transfers, and handles reverts and returned `false` values in a consistent way.

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

There are two ways to call another contract:

* Define an interface and call the function through that interface.
* Create the call manually with the [Application Binary Interface (ABI)](https://docs.soliditylang.org/en/v0.8.3/abi-spec.html). The authors of this code chose this approach.

```solidity
        require(
            success && (data.length == 0 || abi.decode(data, (bool))),
            'TransferHelper::safeApprove: approve failed'
        );
    }
```

For backward compatibility with older ERC20 tokens, an ERC20 call can fail in two ways: the call can revert, in which case `success` is `false`, or the call can succeed but return `false`, in which case there is output data that decodes to the Boolean value `false`.

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

This function implements the [ERC20 transfer operation](https://eips.ethereum.org/EIPS/eip-20#transfer), which transfers tokens from the caller to another account.

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

This function implements the [ERC20 `transferFrom` operation](https://eips.ethereum.org/EIPS/eip-20#transferfrom), which allows one account to spend an allowance provided by another account.

```solidity
    function safeTransferETH(address to, uint256 value) internal {
        (bool success, ) = to.call{value: value}(new bytes(0));
        require(success, 'TransferHelper::safeTransferETH: ETH transfer failed');
    }
}
```

This function transfers ether to an account. Any call to another contract can attempt to send ether. Because the function does not need to call a specific function on the receiver, it does not need to send calldata.

### Conclusion <a href="#conclusion" id="conclusion"></a>

This source-code walkthrough is fairly long, roughly around 50 pages in the original article. If you have read this far, you now have a better understanding of the considerations involved in writing real applications, rather than only small examples, and should be better prepared to write contracts for your own use cases.

Now go build something useful, and hopefully something surprising.
