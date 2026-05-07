# Uniswap V2 Libraries

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
