# Uniswap V2 Factory and ERC20

#### UniswapV2Factory.sol <a href="#uniswapv2factory" id="uniswapv2factory"></a>

[This contract](https://github.com/Uniswap/uniswap-v2-core/blob/master/contracts/UniswapV2Factory.sol) implements pair creation for token swaps.

```solidity
pragma solidity =0.5.16;

import './interfaces/IUniswapV2Factory.sol';
import './UniswapV2Pair.sol';

contract UniswapV2Factory is IUniswapV2Factory {
    address public feeTo;
    address public feeToSetter;
```

These state variables are required for protocol fees. See page 5 of the [whitepaper](https://uniswap.org/whitepaper.pdf). The `feeTo` address receives the liquidity pool tokens that represent accrued protocol fees, while `feeToSetter` is the address allowed to change `feeTo` to a different address.

```solidity
    mapping(address => mapping(address => address)) public getPair;
    address[] public allPairs;
```

These variables track pairs, meaning the exchanges between two tokens.

The first variable, `getPair`, is a mapping that identifies a pair contract by the two ERC20 tokens being exchanged. ERC20 tokens are identified by the addresses of their implementing contracts, so both keys and values are addresses. To obtain the pair address for swapping from `tokenA` to `tokenB`, use `getPair[<tokenA address>][<tokenB address>]` or the reverse order.

The second variable, `allPairs`, is an array containing the addresses of all pairs created by this factory. In Ethereum, a contract cannot iterate over the contents of a mapping or retrieve a list of all keys. Therefore, this array is the only way to know which exchanges are managed by this factory.

Note: the reason you cannot iterate over all mapping keys is that contract storage is _very expensive_, so contracts should use and modify as little storage as possible. You can create [iterable mappings](https://github.com/ethereum/dapp-bin/blob/master/library/iterable\_mapping.sol), but they require storing an additional key list. Most applications do not need that extra storage.

```solidity
    event PairCreated(address indexed token0, address indexed token1, address pair, uint);
```

This event is emitted when a new pair is created. It includes the token addresses, the pair address, and the total number of exchange pairs managed by the factory.

```solidity
    constructor(address _feeToSetter) public {
        feeToSetter = _feeToSetter;
    }
```

The constructor only specifies `feeToSetter`. The factory starts without protocol fees, and only `feeSetter` can change that configuration.

```solidity
    function allPairsLength() external view returns (uint) {
        return allPairs.length;
    }
```

This function returns the number of trading pairs.

```solidity
    function createPair(address tokenA, address tokenB) external returns (address pair) {
```

This is the main function of the factory. It creates a pair between two ERC20 tokens. Notice that anyone can call this function. Creating a new pair does not require permission from Uniswap.

```solidity
        require(tokenA != tokenB, 'UniswapV2: IDENTICAL_ADDRESSES');
        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
```

We want the address of the new exchange contract to be deterministic, so it can be precomputed off-chain. This is useful, for example, for [layer-2 transactions](https://ethereum.org/en/developers/docs/scaling/). To make this possible, the token addresses must always be ordered consistently, regardless of the order in which the function receives them.

```solidity
        require(token0 != address(0), 'UniswapV2: ZERO_ADDRESS');
        require(getPair[token0][token1] == address(0), 'UniswapV2: PAIR_EXISTS'); // single check is sufficient
```

Large liquidity pools are preferable to small liquidity pools because their prices are more stable. For each token pair, we do not want multiple liquidity pools. If a pair already exists, there is no need to create another pair for the same two tokens.

```solidity
        bytes memory bytecode = type(UniswapV2Pair).creationCode;
```

To create a new contract, we need the creation code, which includes the constructor and the code that writes the actual Ethereum Virtual Machine bytecode into storage. In Solidity, this is usually written as `addr = new <name of contract>(<constructor parameters>)`, and the compiler handles the details. However, to obtain a deterministic contract address, the factory needs the [CREATE2 opcode](https://eips.ethereum.org/EIPS/eip-1014). When this code was written, Solidity did not yet support this opcode directly, so the contract manually obtains the bytecode. This is no longer a problem because [Solidity now supports CREATE2](https://docs.soliditylang.org/en/v0.8.3/control-structures.html#salted-contract-creations-create2).

```solidity
        bytes32 salt = keccak256(abi.encodePacked(token0, token1));
        assembly {
            pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
        }
```

When Solidity did not support the opcode directly, it could be invoked through [inline assembly](https://docs.soliditylang.org/en/v0.8.3/assembly.html).

```solidity
        IUniswapV2Pair(pair).initialize(token0, token1);
```

The factory calls `initialize` to tell the newly created exchange which two tokens it can swap.

```solidity
        getPair[token0][token1] = pair;
        getPair[token1][token0] = pair; // populate mapping in the reverse direction
        allPairs.push(pair);
        emit PairCreated(token0, token1, pair, allPairs.length);
    }
```

The factory stores the new pair information in its state variables and emits an event to notify the outside world that a new pair contract has been created.

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

These two functions allow `setFeeTo` to manage the protocol-fee recipient, if fees are enabled, and allow `setFeeToSetter` to be changed to a new address.

#### UniswapV2ERC20.sol <a href="#uniswapv2erc20" id="uniswapv2erc20"></a>

[This contract](https://github.com/Uniswap/uniswap-v2-core/blob/master/contracts/UniswapV2ERC20.sol) implements the ERC20 liquidity token. It is similar to the [OpenZeppelin ERC20 contract](https://ethereum.org/en/developers/tutorials/erc20-annotated-code/), so this note only explains the different part: the `permit` functionality.

Transactions on Ethereum consume ether (ETH), which functions as the native currency of the network. If you have ERC20 tokens but no ether, you cannot send transactions, and therefore you cannot do anything with those tokens on-chain. One solution to this problem is [meta-transactions](https://docs.uniswap.org/protocol/V2/guides/smart-contract-integration/supporting-meta-transactions/). The token owner signs a transaction authorizing someone else to move tokens on-chain, sends that authorization through the network, and the recipient, who has ether, submits the permit on behalf of the owner.

```solidity
    bytes32 public DOMAIN_SEPARATOR;
    // keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)");
    bytes32 public constant PERMIT_TYPEHASH = 0x6e71edae12b1b97f4d1f60370fef10105fa2faae0126114a169c64845d6126c9;
```

This hash is the [identifier for this transaction type](https://eips.ethereum.org/EIPS/eip-712#rationale-for-typehash). Here, the only supported typed message is `Permit` with these parameters.

```solidity
    mapping(address => uint) public nonces;
```

The recipient cannot forge a digital signature. However, the same transaction can be submitted twice, which is a form of [replay attack](https://wikipedia.org/wiki/Replay\_attack). To prevent this, the contract uses a [nonce](https://wikipedia.org/wiki/Cryptographic\_nonce). If the nonce of a new `Permit` is not the previous nonce plus one, the permit is considered invalid.

```solidity
    constructor() public {
        uint chainId;
        assembly {
            chainId := chainid
        }
```

This code obtains the [chain identifier](https://chainid.network/). It uses the Ethereum Virtual Machine's intermediate language, [Yul](https://docs.soliditylang.org/en/v0.8.4/yul.html). Note that in current Yul syntax, `chainid()` should be used instead of `chainid`.

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

This computes the EIP-712 [domain separator](https://eips.ethereum.org/EIPS/eip-712#rationale-for-domainseparator).

```solidity
    function permit(address owner, address spender, uint value, uint deadline, uint8 v, bytes32 r, bytes32 s) external {
```

This function implements approval through a signed permit. It receives the relevant fields as parameters, along with the three scalar values of the [digital signature](https://yos.io/2018/11/16/ethereum-signatures/): `v`, `r`, and `s`.

```solidity
        require(deadline >= block.timestamp, 'UniswapV2: EXPIRED');
```

The transaction must not be accepted after the deadline.

```solidity
        bytes32 digest = keccak256(
            abi.encodePacked(
                '\x19\x01',
                DOMAIN_SEPARATOR,
                keccak256(abi.encode(PERMIT_TYPEHASH, owner, spender, value, nonces[owner]++, deadline))
            )
        );
```

`abi.encodePacked(...)` is the message we expect to receive. The contract already knows what the nonce should be, so the nonce does not need to be supplied as an independent trusted value.

The Ethereum signing algorithm expects a 256-bit value to sign, so the contract uses the `keccak256` hash function.

```solidity
        address recoveredAddress = ecrecover(digest, v, r, s);
```

From the digest and the signature, the contract can compute the signing address with [ecrecover](https://coders-errand.com/ecrecover-signature-verification-ethereum/).

```solidity
        require(recoveredAddress != address(0) && recoveredAddress == owner, 'UniswapV2: INVALID_SIGNATURE');
        _approve(owner, spender, value);
    }
```

If everything is valid, the permit is treated as an [ERC20 approval](https://eips.ethereum.org/EIPS/eip-20#approve).
