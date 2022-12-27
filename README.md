# celo-sdk-java

## Introduction

Celo-sdk-java, originally adapted from Ethereum web3j, is a Java library for working with the Celo Blockchain and Celo Core Contracts.

## Features

- Connect to a node
- Access web3 object to interact with node's Json RPC API
- Send Transaction with celo's extra fields: (feeCurrency)
- Simple interface to interact with CELO and cUSD
- Simple interface to interact with Celo Core contracts
- Utilities

## Getting Started

### Prerequisites
Java 11  
Gradle 7

### Install
Install from repositories:  
maven  
```
<dependency>
    <groupId>io.github.gconnect</groupId>
    <artifactId>celo-sdk-java</artifactId>
    <version>0.0.2</version>
</dependency>

```
build.gradle
```
implementation 'io.github.gconnect:celo-sdk-java:0.0.2'
```
[Celo SDK Java Maven Central Dependency](https://central.sonatype.dev/artifact/io.github.gconnect/celo-sdk-java/0.0.2)

#### Install manually
If you want to generate the jar and import manually.
```
git clone hhttps://github.com/gconnect/celo-sdk-java.git
./gradlew clean build -xtest
./gradlew publishToMavenLocal
```

In your settings.gradle file when working on an android project if installing it manually add `mavenLocal()` inside the dependencyResolutionManagement like this 👇

```
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        mavenLocal()
    }
}
```

Inside build.gradle file add the below code in the dependencies
```
implementation 'org.celo:contractkit:0.0.1'
```




#### Testing

Most of tests uses Ganache testnet. 

To start devchain run `./scripts/start_devchain.sh` script.

To reset devchain to the initial state run `./scripts/reset_devchain.sh` script. It's recommend to run script before run all tests. 

```
./scripts/reset_devchain.sh
./scripts/start_devchain.sh

./gradlew clean test
```

#### Publishing to Sonatype Maven Central
You can read more about it [here](https://central.sonatype.org/publish/publish-gradle/#jar-files)

1. Create an Account on [Sonatype]([https://bintray.com/](https://auth.sonatype.com/auth/realms/sonatype/login-actions/authenticate?client_id=lift&tab_id=SMPHIGZkjy0))
2. Create an issue and an empty repository with the Sonatype issue ID
3. Follow the steps in 1 above to update your build.gradle file
4. Build and upload library

```
./gradlew publish
```

5. Once published go the [Nexus Repository Manager](https://s01.oss.sonatype.org/#stagingRepositories). Select Staging repositories. Your published package will be shown here. Then click close. If everything checks out correctly the release button will be clickable for you to release the package to the public.

6 - It will take a while maybe 30 minutes for the publised package to be available on [Maven Central Repository](https://central.sonatype.dev/)

### Initializing the ContractKit

To start working with ContractKit you need a `kit` instance and a valid net to connect with. In this example will use `alfajores` (you can read more about it [here](../../getting-started/alfajores-testnet.md))

```java
import org.celo.contractkit.ContractKit; 

ContractKit contractKit = ContractKit.build(new HttpService("https://alfajores-forno.celo-testnet.org"));
```

#### Initialize the ContractKit with your own node

If you are hosting your own node you can connect our ContractKit to it.

Same as `Web3` we support `WebSockets`, `RPC` and connecting via `IPC`.
For this last one you will have to initialize the `kit` with an instance of `Web3` that has a **valid** `IPC Provider`

```java
import org.celo.contractkit.ContractKit; 
import org.web3j.protocol.Web3j;
import org.web3j.protocol.ipc.UnixIpcService;

Web3j web3j = Web3j.build(new UnixIpcService("/path/to/socketfile"));
ContractKit contractKit = new ContractKit(web3j);
```

## ContractKit Usage

The following are some examples of the capabilities of the `ContractKit`, assuming it is already connected to a node. If you aren't connected, [here is a refresher.](../walkthroughs/hellocontracts.md#deploy-to-alfajores)

### Setting Default Tx Options

`kit` allows you to set default transaction options:

```java
Web3j web3j = Web3j.build(new HttpService(ContractKit.ALFAJORES_TESTNET));

ContractKitOptions config = new ContractKitOptions.Builder()
    .setFeeCurrency(CeloContract.GoldToken)
    .setGasPrice(BigInteger.valueOf(21_000))
    .build();
ContractKit contractKit = new ContractKit(web3j, config);
```

Multiple accounts can be added to the kit wallet. The first added account will be used by default.
```java
Credentials credentials = Credentials.create(somePrivateKey);
contractKit.addAccount(credentials);

or

contractKit.addAccount(somePrivateKey);
```

To change default account to sign transactions 
```java
contractKit.setDefaultAccount(publicKey);
```

### Getting the Total Balance

This method from the `kit` will return the CELO, locked CELO, cUSD and total balance of the address

```java
AccountBalance balance = contractKit.getTotalBalance(myAddress);
```

### Deploy a contract

Deploying a contract with the default account already set. 

You can verify the deployment on the [Alfajores block explorer here](https://alfajores-blockscout.celo-testnet.org/). Wait for the receipt and log it to get the transaction details.

```java
Web3j web3j = Web3j.build(new HttpService(ContractKit.ALFAJORES_TESTNET));

ContractKit contractKit = new ContractKit(web3j);
contractKit.addAccount(PRIVATE_KEY);

GoldToken deployedGoldenToken = contractKit.contracts.getGoldToken().deploy().send();

TransactionReceipt receipt = deployedGoldenToken.getTransactionReceipt().get();
String hash = receipt.getTransactionHash();
```

### Buying all the CELO I can, with the cUSD in my account

```java
ExchangeWrapper exchange = contractKit.contracts.getExchange();
StableTokenWrapper stableToken = contractKit.contracts.getStableToken();
GoldTokenWrapper goldToken = contractKit.contracts.getGoldToken();

BigInteger cUsdBalance = stableToken.balanceOf(myAddress).send();

TransactionReceipt approveTx = stableToken.approve(exchange.getContractAddress(), cUsdBalance).send();
String approveTxHash = approveTx.getTransactionHash();

BigInteger goldAmount = exchange.quoteUsdSell(cUsdBalance).send();
TransactionReceipt sellTx = exchange.sellDollar(cUsdBalance, goldAmount).send();
String sellTxHash = sellTx.getTransactionHash();
```

## Celo Core Contracts. Wrappers / Registry

### Interacting with CELO & cUSD

celo-blockchain has two initial coins: CELO and cUSD (stableToken).
Both implement the ERC20 standard, and to interact with them is as simple as:

```java
GoldTokenWrapper goldtoken = contractKit.contracts.getGoldToken();
BigInteger goldBalance = goldtoken.balanceOf(someAddress).send();
```

To send funds:

```java
BigInteger oneGold = Convert.toWei(BigDecimal.ONE, Convert.Unit.ETHER).toBigInteger();
TransactionReceipt tx = goldToken.transfer(someAddress, oneGold).send();
String hash = tx.getTransactionHash();
```

To interact with cUSD, is the same but with a different contract:
```java
StableTokenWrapper stableToken = contractKit.contracts.getStableToken();
```

### Interacting with Other Celo Contracts

Apart from GoldToken and StableToken, there are many core contracts.

For the moment, we have contract wrappers for:

- Accounts
- Attestations
- BlockchainParameters
- DobleSigningSlasher
- DowntimeSlasher
- Election
- Escrow
- Exchange (Uniswap kind exchange between Gold and Stable tokens)
- GasPriceMinimum
- GoldToken
- Gobernance
- LockedGold
- Reserve
- SortedOracles
- Validators
- StableToken

### A Note About Contract Addresses

Celo Core Contracts addresses, can be obtained by looking at the `Registry` contract.
That's actually how `kit` obtain them.

We expose the registry api, which can be accessed by:

```java
String goldTokenAddress = contractKit.contracts.addressFor(CeloContract.GoldToken);
```

### Sending Custom Transactions

Celo transaction object is not the same as Ethereum's. There are three new fields present:

- feeCurrency (address of the ERC20 contract to use to pay for gas and the gateway fee)
- gatewayFeeRecipient (coinbase address of the full serving the light client's trasactions)
- gatewayFee (value paid to the gateway fee recipient, denominated in the fee currency)

This means that using `web3.eth.sendTransaction` or `myContract.methods.transfer().send()` should be **avoided**.

Instead, `kit` provides an utility method to send transaction in both scenarios. **If you use contract wrappers, there is no need to use this.**

For a raw transaction:
```java
CeloRawTransaction tx = CeloRawTransaction.createCeloTransaction(
                BigInteger.ZERO,
                GAS_PRICE,
                GAS_LIMIT,
                someAddress,
                oneGwei,
                contractKit.contracts.addressFor(CeloContract.StableToken),
                null,
                null
        );
EthSendTransaction receipt = contractKit.sendTransaction(tx);
assertNotNull(receipt.getTransactionHash());
```
