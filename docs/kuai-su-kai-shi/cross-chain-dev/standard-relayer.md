# 标准中继器

![标准中继器](../../.gitbook/assets/auto-relayer.png)

标准中继器为一个链上合约提供了一种机制，使其能够向另一条链上的合约发送消息，开发者无需处理任何链下部署。

{% hint style="warning" %}
目前，标准中继器功能仅限于 EVM 环境。

点击[此处](../../blockchain-environments/evm/)查看 EVM 环境区块链的完整列表。
{% endhint %}

## 教程

* [Hello Wormhole](../../tutorials/quick-start/) 一个涵盖跨 EVM 生态的信息传递的教程
* [Hello Token](../tutorials/hello-token.md) 一个涵盖跨 EVM 生态的代币转移的教程

## 链上

在链上，智能合约通过 [IWormholeRelayer](https://github.com/wormhole-foundation/wormhole-relayer-solidity-sdk/blob/main/src/interfaces/IWormholeRelayer.sol) 进行发送和接收交互。

### 发送信息

要向另一条 EVM 链上的合约发送信息，我们可以调用 `IWormholeRelayer` 接口提供的 `sendPayloadToEvm` 方法。

```solidity
function sendPayloadToEvm(
    // Chain ID in Wormhole format
    uint16 targetChain,     
    // Contract Address on target chain we're sending a message to
    address targetAddress,  
    // The payload, encoded as bytes
    bytes memory payload,   
    // How much value to attach to the delivery transaction 
    uint256 receiverValue,  
    // The gas limit to set on the delivery transaction
    uint256 gasLimit        
) external payable returns (
    // Unique, incrementing ID, used to identify a message
    uint64 sequence
);
```

`sendPayloadToEvm` 方法被标记为 `payable` ，因此我们可以为我们的交易付款，使其能够被提交。

通过调用 `quoteEVMDeliveryPrice` 来确定附加到调用的值，它提供了目标链上 gas 成本的估计值。

```solidity
function quoteEVMDeliveryPrice(
    // Chain ID in Wormhole format
    uint16 targetChain,
    // How much value to attach to delivery transaction 
    uint256 receiverValue,
    // The gas limit to attach to the delivery transaction
    uint256 gasLimit
) external view returns (
    // How much value to attach to the send call
    uint256 nativePriceQuote, 
    // 
    uint256 targetChainRefundPerGasUnused
);
```

此方法应该在发送信息之前调用，并且应该在调用发送有效负载时，将返回的nativePriceQuote值添加到其中，以支付目标链上的交易成本。

总的来说，跨 EVM 链发送信息可以很简单：

```solidity
// Get a quote for the cost of gas for delivery
(cost, ) = wormholeRelayer.quoteEVMDeliveryPrice(
    targetChain,
    valueToSend,
    GAS_LIMIT
);

// Send the message
wormholeRelayer.sendPayloadToEvm{value: cost}(
    targetChain,
    targetAddress,
    abi.encode(payload),
    valueToSend, 
    GAS_LIMIT
);
```

### 接收信息

要使用 `Standard Relayer` 功能接收信息， 目标合约必须实现 [IWormholeReceiver](https://github.com/wormhole-foundation/wormhole-relayer-solidity-sdk/blob/main/src/interfaces/IWormholeReceiver.sol) 接口。

```solidity
function receiveWormholeMessages(
    bytes memory payload,           // Message passed by source contract 
    bytes[] memory additionalVaas,  // Any additional VAAs that are needed (Note: these are unverified) 
    bytes32 sourceAddress,          // The address of the source contract
    uint16 sourceChain,             // The Wormhole chain ID
    bytes32 deliveryHash            // A hash of contents, useful for replay protection
) external payable;
```

函数内部的逻辑可以是执行针对特定有效负载所需的任何业务逻辑。

### 其它注意事项

在开发过程中应考虑一些实现细节，以确保安全和改善用户体验。

* 从中继器接收信息
  * 检查预期的发射器
  * 在任何 additionalVAAs上调用 parseAndVerify&#x20;
* 重放保护
* 信息有序性
  * 不保证发送信息的顺序
* 链式转发/调用
* 退还多付的 gasLimit
* 退还多付的金额

## 链下

如果利用自动中继功能，则不需要实现链下逻辑。

虽然不需要链下程序，但开发人员可能希望跟踪传输中的信息的进度。要跟踪传输中消息的进度，请使用 worm CLI  工具的 `status`  子命令。

```sh
$ worm status mainnet ethereum 0xdeadbeef
```

请参阅 [CLI 工具文档](../../reference/cli-docs/)了解安装和使用方法。

## 另请参阅

有关 EVM 链的参考文档，请点击[此处](../../blockchain-environments/evm/) 。
