# Uniswap V2 Factory and ERC20

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

我们希望新兑换的地址可以确定， 这样它可以在链下预计算 （这对于[第二层的交易](https://ethereum.org/en/developers/docs/scaling/) 来说比较有用）。 为了做到这一点，我们需要代币地址始终按顺序排列，无论收到代币地址的顺序如何， 都需要在这里排序。

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

[本合约](https://github.com/Uniswap/uniswap-v2-core/blob/master/contracts/UniswapV2ERC20.sol)实现了 ERC-20 流动代币。 这与 [OpenZeppelin ERC-20 合约](https://ethereum.org/en/developers/tutorials/erc20-annotated-code/)相似，因此 这里仅解释不同的部分，`permit` 的功能。

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
