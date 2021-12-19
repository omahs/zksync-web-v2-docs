# Getting started

## Concept

While most of the exisitng SDKs should work out of the box, deploying smart contracts or using unique zkSync features like paying fees in other tokens requires providing additional fields to those that the Ethereum transactions have by default. 

To provide easy access to all of the features of zkSync 2.0, we created `zksync-web3` JavaScript SDK, which is made in a way that is has interface very similar to those of [ethers](https://docs.ethers.io/v5/). In fact, `ethers` is a peer dependency of the library and most of the objects exported by `zksync-web3` (e.g. `Wallet`, `Provider` etc) inherit from the corresponding `ethers` objects and override only the fields that need to be changed.

The library is made in such a way that changing `ethers` with `zksync-web3` in your imports should be sufficient in most of the cases.  

## Adding dependencies

```bash
yarn add zksync-web3
yarn add ethers@5 # ethers is a peer dependency of zksync-web3
```

Then you can import all the content of the `ethers` library and zkSync library with the following statement:

```typescript
import * as zksync from "zksync-web3";
import * as ethers from "ethers";
```

## Connecting to zkSync network

To interact with zkSync network users need to know the endpoint of the operator node.

```typescript
// Currently, only one environment is supported.
// Please do not share the link outside of your team. 
// We plan to gradually roll out the network to wider audience.
const syncProvider = new zksync.Provider("https://z2-dev-api.zksync.dev"); 
```

**Note:** Currently, only `rinkeby` network is supported.

Some operations require access to the Ethereum network. We use `ethers` library to interact with
Ethereum.

```typescript
const ethProvider = ethers.getDefaultProvider("rinkeby");
```

## Creating a Wallet

To control your account in zkSync, use the `zksync.Wallet` object. It can sign transactions with keys stored in
`ethers.Wallet` and send transaction to zkSync network using `zksync.Provider`.

```typescript
// Derive zksync.Wallet from ethereum private key.
// zkSync's wallets support all of the methods of ethers' wallets.
// Also, both providers are optional and can be connected to later via `connect` and `connectToL1`.
const syncWallet = new zksync.Wallet(PRIVATE_KEY, syncProvider, ethProvider);
```

## Depositing funds

We are going to deposit `1.0 ETH` to our zkSync account.

```typescript
const deposit = await syncWallet.deposit({
  token: zksync.utils.ETH_ADDRESS,
  amount: ethers.utils.parseEther("1.0"),
});
```

**NOTE:** Each token inside zksync has an address. For ERC20 tokens this address coincides with the address in L1, with
the exception of ETH, the address `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` is used for it. To get the ETH address in
zkSync, we can use the constant `ETH_ADDRESS`.

After the tx is submitted to the Ethereum node, we can track its status using the transaction handle:

```typescript
// Await processing of the deposit on L1
const ethereumTxReceipt = await deposit.waitL1Commit();

// Await processing the deposit on zkSync
const depositReceipt = await deposit.wait();
```

## Checking zkSync account balance

```typescript
// Retreiving the current (committed) balance of an account
const committedEthBalance = await syncWallet.getBalance(zksync.utils.ETH_ADDRESS);

// Retrieving the balance of an account in the last finalized block
const finalizedEthBalance = await syncWallet.getBalance(ETH_ADDRESS, "finalized");
```

You can read more about what are committed and finalized balances [here](). <!--- TODO: link to the Celeste part of the docs -->

## Performing a transfer

Now, let's create a second wallet and transfer some funds into it. Note that we can send assets to any fresh Ethereum
account, without preliminary registration!

```typescript
const syncWallet2 = new zksync.Wallet(PRIVATE_KEY2, syncProvider, ethProvider);
```

We are going to transfer `1 ETH` to another account and pay `1 DAI` as a fee to the operator (zkSync account balance of the sender is going to be decreased by `1 ETH` and `1 DAI`).

**NOTE:** You can use any token inside zksync if it was previously added via the `addToken` operation. If the token does not exist, any user can add it!

```typescript
const amount = ethers.utils.parseEther("1.0");
// get amount in wei by the decimals parameter for the token
const fee = ethers.utils.parseUnits("1.0", 18);

const transfer = await syncWallet.transfer({
  to: syncWallet2.address,
  token: zksync.utils.ETH_ADDRESS,
  amount,
  feeToken: "0x6b175474e89094c44da98b954eedeac495271d0f", // DAI address
  fee,
});
```

**Note:** that setting fee and fee token manually is not required. If `feeToken` field is omitted, then the `token` will
be used to pay the fee. If `fee` field is omitted, SDK will choose the lowest possible fee acceptable by server:

```typescript
const amount = ethers.utils.parseEther("1.0");

const transfer = await syncWallet.transfer({
  to: syncWallet2.address,
  token: zksync.utils.ZKSYNC_ADDRESS,
  amount,
});
```

To track the status of this transaction:

```typescript
// Await commitment
const transferReceipt = await transfer.wait();

// Await finalization on L1
const transferReceipt = await transfer.waitFinalize();
```

## Withdrawing funds

There are two ways to withdraw funds from zkSync to Ethereum, calling the operation through L2 or through L1. If the
withdrawal operation was called through L1, then the operator will have a period of time during which he must process
the transaction, otherwise `PriorityMode` will be turned on. This ensures that the operator cannot stage the
transaction. But in most cases, a call via L2 is sufficient.

```typescript
const withdrawL2 = await syncWallet.withdraw({
  token: zksync.utils.ETH_ADDRESS,
  amount: ethers.utils.parseEther("0.5"),
});
```

Assets will be withdrawn to the target wallet after the zero-knowledge proof of zkSync block with this operation is
generated and verified by the mainnet contract.

We can wait until ZKP verification is complete:

```typescript
await withdrawL2.waitFinalize();
```

## Deploying a contract

First you need to select the bytecode for the contract. The easiest way is to read it from a file, so we will use the
`fs` library, but in general, you can use any way you like.

```typescript
import fs from "fs";

// Select the path to the file with the bytecode.
const bytecodePath = "<PATH>";
const bytecode = new Uint8Array(fs.readFileSync(bytecodePath));
```

Next, let's select the type of account for the contract. There are two options: `ZkRollup` and `ZkPorter`. The first one
provides ultra-security (as L1) and cheap fees for interacting with it, and the second one offers security and ultra
cheap fees. Different account types are suitable for different purposes, but for example, let's stick with the
`ZkPorter`.

```typescript
const accountType = AccountType.ZkPorter;
```

Finally, we can deploy the contract!

```typescript
const deployContract = await syncWallet.deployContract({
  accountType,
  bytecode,
  feeToken: zksync.utils.ETH_ADDRESS,
  fee: ethers.utils.parseEther("0.001"),
});
```

**Note:** that setting fee manually is not required. If `fee` field is omitted, SDK will choose the lowest possible fee
acceptable by server:

```typescript
const deployContract = await syncWallet.syncDeployContract({
  accountType,
  bytecode,
  feeToken: zksync.utils.ETH_ADDRESS,
});
```

To track the status of this transaction:

```typescript
const deployContractReceipt = await deployContract.wait();

// If the contract has been successfully deployed, then with `deployContractReceipt`
// we can get the address of the deployed contract.
const contractAddress = deployContractReceipt.contractAddress;

if (!contractAddress) {
  throw new Error("Failed getting the address of the deployed contract");
}
```

### Calling a contract

For calling contracts we can use the `.rawExecute` method of the wallet, but it is only useful if you want to provide
raw calldata for the contract call.

There also exists a more convenient interface in the form of `zksync.Contract` class which works almost exactly like the
one in `ethers`: just call a method with its arguments, don't forget to specify a `feeToken`.

Let's call a method `test` with parameter `42` on the contract we deployed in the previous section, using ETH to pay the
fees.

```typescript
const contract = new zksync.Contract(contractAddress, abi, syncWallet);
// You can load contract's ABI from JSON as well
const abi = ["function test(uint256) public"];
const feeToken = zksync.utils.ETH_ADDRESS;
// We must specify feeToken for non-readonly calls as the last argument.
// We can also specify nonce, timerange etc.
const contractCall = await contract.test(42, { feeToken });
const callReceipt = await contractCall.wait();
```

## Add new token

Adding tokens to zksync is completely permissionless, any user can add any ERC20 token from Ethereum to zkSync as the
first class citizen token. After adding a token, it can be used in all types of transactions.

Make sure you are connected to an L1 provider to use this method.

```typescript
const tokenAddress = "0x...";
const addToken = await syncWallet.addToken(tokenAddress);
```