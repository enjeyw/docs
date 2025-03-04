---
description: Developing a cross-blockchain application with the arbitrary message bridge
---

# How to develop a cross-blockchain application using the AMB

{% hint style="info" %}
The complete AMB bridge interface can be found [here](https://github.com/poanetwork/poa-bridge-contracts/blob/master/contracts/interfaces/IAMB.sol)
{% endhint %}

## Using AMB exposed methods

### Call a method in another chain using the AMB bridge

AMB is about the invocation of a contract's method in another chain. **A contract on one side must know a method of a contract on another side**. The method name and parameters are encoded and passed to the `requireToPassMessage` method of the bridge contract.

```javascript
function requireToPassMessage(address _contract, bytes _data, uint256 _gas) external;
```

* `address _contract`  address of the contract on the other network.
* `bytes _data`encoded bytes of the method selector and the parameters that will be called in the contract on the other network. In the [Code Examples](how-to-develop-xchain-apps-by-amb.md#code-examples) section there is an example of how to generate this parameter.
* `uint256 _gas` The gas to be provided in execution of the method call in the contract on the other network. To generate this parameter you need to measure the gas usage of the method to be executed. It can be done by manual invocation of the method in a dev environment or by using a tool like [eth-gas-reporter](https://github.com/cgewecke/eth-gas-reporter) to get the gas usage of the methods from the unit tests.

  The AMB bridge provides a maximum gas value to be provided in the execution of the method, so the provided gas value must not exceed this limit. To get this limit, you can call the method: 

  ```javascript
  function maxGasPerTx() external view returns (uint256);
  ```

### Receive a method call from the AMB bridge

If the contract receiving a method call from the AMB bridge needs to perform some critical actions, it is wise to consider the following security measures:

* Check that the `msg.sender` is the address of the bridge.
* Check the address of the invoking contract from the other side. To do this, the contract can call the method `messageSender()` from the AMB bridge to know who generated the message that is defined as:

  ```javascript
  function messageSender() external view returns (address);
  ```

  Another useful method from the AMB Bridge is the `transactionHash()` method. It returns the hash of the transaction that caused the invocation of `requireToPassMessage` on the other network. 

  ```javascript
  function transactionHash() external view returns (bytes32);
  ```

### Security

Every time `requireToPassMessage` is called, the AMB bridge validators will listen to the generated event and provide their signature to bridge the message. Once enough signatures are collected, the message is marked as processed. Only then does it proceed with the execution of the method call. This guarantees the message will be executed only one time.

### Handling failed messages

It is possible that the call execution of message relayed by the bridge could fail. The reasons could be related to some specific logic of the invoked method, an insufficient gas limit provided to the method call, or invalid data.

The AMB bridge exposes methods to help retrieve information related to the failed message.

```text
function messageCallStatus(bytes32 _txHash) external view returns (bool);
function failedMessageReceiver(bytes32 _txHash) external view returns (address);
function failedMessageSender(bytes32 _txHash) external view returns (address);
function failedMessageDataHash(bytes32 _txHash) external view returns (bytes32);
```

All tmethods accept as a parameter the hash of the transaction that originated the message on the other network.

* `messageCallStatus` returns the result of the message call execution.
* `failedMessageReceiver` returns the address that received the call execution of the message.
* `failedMessageSender` return the address that generated the message on the other network.
* `failedMessageDataHash` return the hash `keccak256(data)` associated to the originating transaction hash. The contract-sender is responsible for providing unique sequence as part of the `data`. Where `data` refers to the `data` parameter in `requireToPassMessage` method.

## Example of ERC677 to ERC677 using AMB bridge

We can use AMB to move ERC677 tokens between two chains. To do this, we'll have two contracts that communicate with each other: contract A receives tokens, locks them and instructs contract B to mint the same number of tokens in the other chain. In the inverse case, contract B receives tokens, burns them and instructs contract A to unlock the burned amount in the other chain.

The implementation of the contracts for this ERC677-TO-ERC677 built on top of the AMB bridge can be found [here](https://github.com/poanetwork/poa-bridge-contracts/tree/master/contracts/upgradeable_contracts/amb_erc677_to_erc677).

In this implementation we have:

* A Token Management contract on Foreign side that will lock/unlock transferred tokens and send requests to Mint tokens on Home side.
* A Token Management contract on Home side that will Mint/Burn transferred tokens and send requests to Unlock tokens on Foreign side.

Example of Home Token Management contract tested in Sokol:

* [Proxy](https://blockscout.com/poa/sokol/address/0x6feaEfd9F6d88178579e0F2fd01E511f948d21E3)
* [Implementation](https://blockscout.com/poa/sokol/address/0xa08d168cc4fdab5d17161417807aa913ad1c9203) 

Example of Foreign Token Management contract tested in Kovan

* [Proxy](https://blockscout.com/eth/kovan/address/0x65ea3665105Ea69b62bd1d4741Fa12561473eD08)
* [Implementation](https://blockscout.com/eth/kovan/address/0xf1ac8de3213be1f3ee2c55b752be803fde7b25cd)

Deployed bridges contracts are available [here](https://forum.poa.network/t/using-arbitrary-message-bridging/2710/8).

![AMB-ERC677-ERC677](https://i.imgur.com/IAIr4YO.png)

### Token transfer flow

In the case when a user has Tokens on the Foreign side and wants to bridge them to the Home network: 1. The user calls the method `transferAndCall` of the token contract with the value and the foreign token management contract address as target. 2. The tokens are transferred and the token contract calls `onTokenTransfer` method of the token management contract. 3. In `onTokenTransfer` method, the token management contract calls `requireToPassMessage` method of Foreign AMB bridge contract with parameters indicating that method `handleBridgedTokens` of the Home token management contract should be called with the recipient and value parameters of the token transfer.

Then, when the AMB bridge process the message, on Home network: 1. The AMB Oracle will call Home AMB bridge contract. 1. Home AMB bridge will call `handleBridgedTokens` method of the Home Token Management contract. 2. `handleBridgedTokens` method will Mint the Tokens.

Here is a representation of the steps explained above:

![AMB-ERC677-ERC677-Transfer](https://i.imgur.com/LGDqqkp.png)

Transferring tokens from Home network to the Foreign network works in a similar way. The only difference is that Home Token Management contract will Burn the transferred tokens, and Foreign Token Management contract will unlock the tokens.

## Code examples

Taking in consideration that the token contract address, the AMB bridge contract address, the token management contract address of the other network and the execution gas limit were stored in the contract on the initialization of it, this is an example of `onTokenTransfer` implementation:

```javascript
function onTokenTransfer(address _from, uint256 _value, bytes /*_data*/) external returns (bool) {
    require(msg.sender == erc677token());

    ...

    bytes4 methodSelector = ITokenManagement(address(0)).handleBridgedTokens.selector;
    bytes memory data = abi.encodeWithSelector(methodSelector, _from, _value, uniqueSecuence);
    bridgeContract().requireToPassMessage(tokenManagementContractOnOtherSide(), data, executionGasLimit());

    // Save value and from related to the data hash in case the message fails on the other side
    bytes32 dataHash = keccak256(data);
    setMessageHashValue(dataHash, _value);
    setMessageHashRecipient(dataHash, _from);

    return true;
}
```

Here is the example implementation of the method `handleBridgedTokens` on Home network:

```javascript
function handleBridgedTokens(address _recipient, uint256 _value, bytes32 /* uniqueSecuence */) external {
    require(msg.sender == address(bridgeContract()));
    require(bridgeContract().messageSender() == tokenManagementContract());

    ...

    erc677token().mint(_recipient, _value);
}
```

In case the execution of `handleBridgedTokens` fails, any user could call the following method in Home Network to request a fix for the transfer performed previously on the Foreign Network.

```javascript
function requestFailedMessageFix(bytes32 _txHash) external {
    require(!bridgeContract().messageCallStatus(_txHash));
    require(bridgeContract().failedMessageReceiver(_txHash) == address(this));
    require(bridgeContract().failedMessageSender(_txHash) == tokenManagementContractOnOtherSide());

    // Get the data hash related to the message
    bytes32 dataHash = bridgeContract().failedMessageDataHash(_txHash);

    bytes4 methodSelector = ITokenManagement(address(0)).fixFailedMessage.selector;
    bytes memory data = abi.encodeWithSelector(methodSelector, dataHash);
    bridgeContract().requireToPassMessage(mediatorContractOnOtherSide(), data, requestGasLimit());
}
```

Here is the example implementation of the method in Foreign Network that will unlock the transferred tokens after the request to fix a failed message from the Home Network.

```javascript
function fixFailedMessage(bytes32 _dataHash) external {
    require(msg.sender == address(bridgeContract()));
    require(messageSender() == tokenManagementContractOnOtherSide());
    require(!messageHashFixed(_dataHash));

    // Get values stored in onTokenTransfer method
    address recipient = messageHashRecipient(_dataHash);
    uint256 value = messageHashValue(_dataHash);

    // Mark hash as fixed to avoid fixing it twice
    setMessageHashFixed(_dataHash);

    token().transfer(_recipient, _value);
}
```

{% hint style="success" %}
Instruction is migrated from the POA forum [https://forum.poa.network/t/how-to-develop-a-cross-blockchain-application-by-using-amb-bridge/2963](https://forum.poa.network/t/how-to-develop-a-cross-blockchain-application-by-using-amb-bridge/2963)
{% endhint %}

