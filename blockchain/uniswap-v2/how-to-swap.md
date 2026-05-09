---
description: "Reference: https://medium.com/uv-labs/uniswap-testing-1d88ca523bf0"
---

# Integrating Uniswap V2 Token Swaps

> [**Uniswap**](https://learnblockchain.cn/tags/Uniswap)

This note shows how to integrate with Uniswap V2 to swap tokens, and how to verify the behavior through tests.

![Uniswap & eth](https://img.learnblockchain.cn/pics/20220608144145.jpeg!/scale/60)

In this article, we interact with the real Uniswap V2 contracts, implement a token swap with [Uniswap](https://uniswap.org/), and verify the swap through tests.

Testing smart contract behavior is a good way to gain confidence that the code executes the way we expect, rather than doing something unintended.

In this article, we will also learn how to fork mainnet, impersonate an on-chain account to send transactions, and write tests around that flow.

## About Uniswap V2

Before diving in, let us briefly introduce Uniswap again so that this note is self-contained. Uniswap is a decentralized exchange (DEX) that runs on the Ethereum blockchain, including mainnet and several other networks. As its name suggests, Uniswap is used to trade ERC20 tokens.

Uniswap has three main functions:

* Swap between different tokens.
* Add liquidity to a token pair and receive LP ERC20 liquidity tokens.
* Burn LP ERC20 liquidity tokens and retrieve the paired ERC20 tokens.

In this article, we focus on swapping between different tokens by using a forked mainnet.

## Create and initialize the project

Use the following commands in the CLI to initialize the project:

```bash
mkdir uni_swap && cd uni_swap
npm init -y
```

Install the dependencies required by the project:

```bash
npm install --save hardhat @nomiclabs/hardhat-ethers @nomiclabs/hardhat-waffle ethers @uniswap/v2-core dotenv
```

## Initialize the Hardhat project

To initialize the Hardhat project, run `npx hardhat` in the CLI and create an empty _config.js_ file.

Then customize the Hardhat configuration, because we need to fork mainnet to interact with Uniswap. The Hardhat configuration should look similar to this:

![img](https://img.learnblockchain.cn/pics/20220608144228.png)

Note: replace the `<key>` part in the URL with your own [Alchemy](https://alchemy.com/?r=7d60e34c-b30a-4ffa-89d4-3c4efea4e14b) API key.

## Write the swap contract

Create directories for contracts, scripts, and tests so the code is organized more clearly.

Use the following commands in the CLI to create those directories:

```bash
mkdir contracts && mkdir scripts && mkdir tests
```

To write the swap contract, create a file inside the contracts directory and name it `testSwap.sol`.

In `testSwap.sol`, import the Uniswap-related interfaces and create a contract named **testSwap**.

It should look like this:

![img](https://img.learnblockchain.cn/pics/20220608144250.png)

Now, inside `testSwap`, we need to include the address of the **Uniswap Router**, which we use to perform the token swap.

Use the following code:

```solidity
// Address of the Uniswap V2 router.
address private constant UNISWAP_V2_ROUTER = 0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D;
```

Next, define the function that will be used for the swap:

```solidity
// Swap function.
    function swap (
        address _tokenIn,
        address _tokenOut,
        uint256 _amountIn,
        address _to,
        uint256 _deadline
    ) external {}
```

The function is named **swap**, and it contains the following parameters:

* **\_tokenIn**: the address of the token we want to swap from.
* **\_tokenOut**: the address of the token we want to receive from this transaction.
* **\_amountIn**: the amount of the input token we want to trade.
* **\_to**: the address that receives the output tokens from the swap.
* **\_deadline**: the latest timestamp at which the transaction may be executed. If the deadline is exceeded, the transaction fails.

Inside the swap function, the first thing we need to do is transfer the required amount of _**\_tokenIn**_ from `msg.sender` into the contract:

```solidity
// Transfer tokens from the user to the contract.
IERC20(_tokenIn).transferFrom(msg.sender, address(this), _amountIn);
```

Once this call is executed, **\_amountIn** of **\_tokenIn** is transferred into the `testSwap` contract.

Next, by calling **IERC20** `approve`, we allow the Uniswap contract to spend **\_amountIn** of the token held by the `testSwap` contract.

```solidity
// By calling IERC20 approve, you allow the Uniswap contract to spend the tokens in this contract.
IERC20(_tokenIn).approve(UNISWAP_V2_ROUTER, _amountIn);
```

When swapping through the Uniswap Router, we need to set the **path** for the token swap. The first "hop" in the path is the token we provide, and the last "hop" is the token we expect to receive.

Therefore, we declare an address array named `path` and fill it with the **\_tokenIn** and **\_tokenOut** addresses.

```solidity
address[] memory path;
path = new address[](2);
path[0] = _tokenIn; // DAI
path[1] = _tokenOut; // WETH
```

Next, we call **getAmountsOut** to estimate how many output tokens we can receive. It is useful to know the expected output amount before executing the real swap. The **getAmountsOut** function requires an input amount and an array of token addresses representing the path:

```solidity
uint256[] memory amountsExpected = IUniswapV2Router(UNISWAP_V2_ROUTER).getAmountsOut(
            _amountIn,
            path
);
```

Finally, we call the Uniswap Router function **swapExactTokensForTokens** and pass in the parameters. The first parameter of this function is the exact input amount, `amountIn`; the second parameter is the minimum acceptable output amount, `amountOutMin`. Do not confuse the estimated output array with the input amount semantics.

```solidity
uint256[] memory amountsReceived = IUniswapV2Router(UNISWAP_V2_ROUTER).swapExactTokensForTokens(
            _amountIn,
            (amountsExpected[1]*990)/1000, // Accept 1% slippage.
            path,
            _to,
            _deadline
);
```

The swap contract is now ready.

The complete contract should look similar to this:

![img](https://img.learnblockchain.cn/pics/20220608144306.png)

Use `npx hardhat compile` to check whether the smart contract has any errors.

Now it is time to run tests for the contract.

## Write the test script

Create a file under the _tests_ folder and name it ***`sample-test.js`***.

First, import the ERC20 contract ABI from Uniswap, and define the test structure and the contract addresses we will use.

```javascript
const ERC20ABI = require("@uniswap/v2-core/build/ERC20.json").abi;

describe("Test Swap", function () {
    const DAIAddress = "0x6B175474E89094C44Da98b954EedeAC495271d0F";
    const WETHAddress = "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2";
    const MyAddress = "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B";
    const DAIHolder = "0x5d38b4e4783e34e2301a2a36c39a03c45798c4dd";
}
```

Here we use four addresses:

* **DAIAddress** and **WETHAddress** are the contract addresses of Dai and WETH. They are the tokens used in the trade.
* **MyAddress** is the trader address.
* **DAIHolder** is the address we want to impersonate.

Before writing the test case itself, we deploy the **testSwap** smart contract. We use the following code:

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

Create a test case for the test script and impersonate the **DAIHolder** address defined above.

```javascript
it("should swap", async () => { 
            await hre.network.provider.request({
            method: "hardhat_impersonateAccount",
            params: [DAIHolder],
});
const impersonateSigner = await ethers.getSigner(DAIHolder);
```

In the next step, we use the impersonated account to get its initial **DAI token** balance. Later, we will use this balance for the swap transaction.

We also fetch the **WETH token** balance so that we can observe the result of the token swap.

```javascript
const DAIContract = new ethers.Contract(DAIAddress, ERC20ABI, impersonateSigner)
const DAIHolderBalance = await DAIContract.balanceOf(impersonateSigner.address)

const WETHContract = new ethers.Contract(WETHAddress, ERC20ABI, impersonateSigner)

const myBalance = await WETHContract.balanceOf(MyAddress);
console.log("Initial WETH Balance:", ethers.utils.formatUnits(myBalance.toString()));
```

Then, we use the DAI contract to approve `TestSwap` to spend the amount to be swapped:

```javascript
await DAIContract.approve(TestSwapContract.address, DAIHolderBalance)
```

For the final swap deadline, first get the current timestamp from the latest block:

```javascript
// Get the current timestamp.
const latestBlock = await ethers.provider.getBlockNumber();
const timestamp = (await ethers.provider.getBlock(latestBlock)).timestamp;
```

Execute the transaction by calling the **swap** function we wrote. Pass in the parameters configured above.

This transaction is initiated through **DAIHolder**:

```javascript
await TestSwapContract.connect(impersonateSigner).swap(
            DAIAddress,
            WETHAddress,
            DAIHolderBalance,
            MyAddress,
            timestamp + 1000 // Add 1000 seconds to the current block timestamp.
)
```

Finally, verify the swap transaction:

```javascript
const myBalance_updated = await WETHContract.balanceOf(MyAddress);
console.log("Balance after Swap:", ethers.utils.formatUnits(myBalance_updated.toString()));
const DAIHolderBalance_updated = await DAIContract.balanceOf(impersonateSigner.address);
```

Here, we check the account balances after the swap function has executed.

Below, we write tests to check whether the transaction was actually completed:

```javascript
expect(DAIHolderBalance_updated.eq(BigNumber.from(0))).to.be.true
expect(myBalance_updated.gt(myBalance)).to.be.true;
```

* Because we used the entire balance for the trade, in the first assertion we expect the DAI token balance to equal 0.
* In the second assertion, we check whether the **balance** in our account is greater than it was before.

These are the two tests we want to perform.

`sample-test.js` should look similar to the following. Note the `require` statements at the beginning of the file:

![img](https://img.learnblockchain.cn/pics/20220608144317.png)

Of course, feel free to explore and try more tests with these building blocks.

Now, run the tests with `npx hardhat test`.

The result should look like this:

![img](https://img.learnblockchain.cn/pics/20220608144853.png)

As you can see, the initial balance increases after the swap is completed.

The tests we wrote also pass.

If you followed along to the end, then you have completed the flow.
