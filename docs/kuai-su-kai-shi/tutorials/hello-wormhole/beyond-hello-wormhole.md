# 改进 Hello Wormhole

在第 1 部分（[HelloWormhole](./)），我们编写了一个功能完整的跨链应用程序， 允许用户从一个合约中请求，在不同链上的另一个合约中发出 `GreetingReceived` 事件。

在第 2 部分（[Hello Wormhole 如何工作？](hello-wormhole-explained.md)）中，我们讨论了 Wormhole Relayer 合约在幕后是如何工作的。总之，它通过发布一个带有交付指令的 wormhole 信息来运行，这会提醒交付提供者调用目标链上 Wormhole Relayer 合约的 `deliver` 端点，并最终使用正确的输入调用指定的 `targetAddress` 。

HelloWormhole 是一个很好的示例应用程序，但还有很大的改进空间。让我们讨论一些提高该应用程序安全性和功能的方法！

涵盖的主体：

* 限制发送者
* 退款
* 链式交付
* 交付现有的 VAAs

## 保护功能

### 问题：问候可以来自任何人Issue: The greetings can come from anyone

用户无需通过 HelloWormhole 合约来请求问候——他们可以自己调用 `wormholeRelayer.sendPayloadToEvm{value: cost}(…)`！

举例来说，如果你想在每次请求 `sendCrossChainGreeting` 时都在源 HelloWormhole 合约中存储一些信息，那么这并不理想。

通常情况下，所有请求最好都通过自己的源合同来处理。

**解决方案：**我们可以在 `receiveWormholeMessages` 的实现中检查 `sourceChain` 和 `sourceAddress` 是否是有效的 HelloWormhole 合约，如果不是，则返还。

```solidity
    address registrationOwner;
    mapping(uint16 => bytes32) registeredSenders;

    modifier isRegisteredSender(uint16 sourceChain, bytes32 sourceAddress) {
        require(registeredSenders[sourceChain] == sourceAddress, "Not registered sender");
        _;
    }

    /**
     * Sets the registered address for 'sourceChain' to 'sourceAddress'
     * So that for messages from 'sourceChain', only ones from 'sourceAddress' are valid
     *
     * Assumes only one sender per chain is valid
     * Sender is the address that called 'send' on the Wormhole Relayer contract on the source chain)
     */
    function setRegisteredSender(uint16 sourceChain, bytes32 sourceAddress) public {
        require(msg.sender == registrationOwner, "Not allowed to set registered sender");
        registeredSenders[sourceChain] = sourceAddress;
    }
```

### 问题 1 的解决方案示例

我们在 [Wormhole Solidity SDK](https://github.com/wormhole-foundation/wormhole-solidity-sdk) 中提供了一个基类，其中包含上面 modifier 所示的，使得可以轻松添加这些功能

```solidity
    function receiveWormholeMessages(
        bytes memory payload,
        bytes[] memory, // additionalVaas
        bytes32 sourceAddress,
        uint16 sourceChain,
        bytes32 deliveryHash
    )
        public
        payable
        override
        onlyWormholeRelayer
        isRegisteredSender(sourceChain, sourceAddress)
    {
        latestGreeting = abi.decode(payload, (string));

        emit GreetingReceived(latestGreeting, sourceChain, fromWormholeFormat(sourceAddress));
    }
```

HelloWormhole 仓库中包含了一个[示例合约](https://github.com/wormhole-foundation/hello-wormhole/blob/main/src/extensions/HelloWormholeProtections.sol)（和 [forge 测试](https://github.com/wormhole-foundation/hello-wormhole/blob/main/test/extensions/HelloWormholeProtections.t.sol)），该合同使用这些辅助工具：

你可以将这些辅助工具添加到自己的项目中，如下所示：

```bash
forge install wormhole-foundation/wormhole-solidity-sdk
```

## 功能：接收退款

通常情况下，你无法准确预测你的合同将使用多少 gas。为避免出现“Receiver Failure”（当你的合约重新生效时发生这种情况，例如，因为 gas 用完了），你应该请求一个合理的上限，以确定合同将使用多少 gas。

但是，这意味着，如果我们预计 HelloWormhole 需要花费的 gas 数量在 10000 到 50000 单位之间（均匀随机），那么如果我们每次请求都请求 50000 单位的 gas，就会损失 20000 单位的 gas 预期成本！

幸运的是，`IWormholeRelayer` 接口允许你对目标合约中最终没有使用的任何 gas [进行退款](https://github.com/wormhole-foundation/wormhole/blob/main/ethereum/contracts/interfaces/relayer/IWormholeRelayer.sol#L89)！

```solidity

function sendPayloadToEvm(
    uint16 targetChain,
    address targetAddress,
    bytes memory payload,
    uint256 receiverValue,
    uint256 gasLimit,
    **uint16 refundChain,
    address refundAddress**
) external payable returns (uint64 sequence);
```

如果指定了这些值，则会根据 `refundChain` 和 `targetChain` 的值应用不同的逻辑。

**如果 refundChain 等于 targetChain，**则退款为

```solidity
targetChainRefundPerGasUnused * (gasLimit - gasUsed)
```

将被发送到目标链上的地址 `refundAddress`。

* **gasUsed** 是你的合约（位于 `targetAddress`）在调用 `receiveWormholeMessages` 时使用的 gas 量。

> 注意，必须小于或等于 gasLimit。

* **targetChainRefundPerGasUnused** 是 delivery provider 在交付前引用的常量——这是 `quoteEVMDeliveryPrice` 函数的第二个返回值：

```solidity
function quoteEVMDeliveryPrice(
        uint16 targetChain,
        uint256 receiverValue,
        uint256 gasLimit
) external view returns (uint256 nativePriceQuote, uint256 targetChainRefundPerGasUnused);
```

否则（如果 refundChain 不等于 targetChain）， 那么

1. 将计算在 gas limit 和 receiver 值为 0 的情况下向退款链交付的成本（我们称之为 BASE\_COST）
2. **如果** `TARGET_CHAIN_REFUND = targetChainRefundPerGasUnused * (gasLimit - gasUsed)` **大于 BASE\_COST**，则将执行交付，发送到 `refundChain` 上 `refundAddress` 的 `msg.value` 将是

```solidity
targetChainWormholeRelayer.quoteNativeForChain(refundChain, TARGET_CHAIN_REFUND - BASE_COST, deliveryProviderAddress)
```

> 注意：此处的 deliveryProviderAddress 等于 `targetChainWormholeRelayer.quoteDefaultDeliveryProvider`

HelloWormhole 仓库中包含了一个使用该退款功能的[示例合约](https://github.com/wormhole-foundation/hello-wormhole/blob/main/src/extensions/HelloWormholeRefunds.sol)（和 [forge 测试](https://github.com/wormhole-foundation/hello-wormhole/blob/main/test/extensions/HelloWormholeRefunds.t.sol)）。

## 功能：从 A 链 → B 链 → C 链

假设你希望请求从 A 链 向 B 链传输信息，然后在 B 链传输完成后，你希望向 C 链传输一些信息。

其中一种方法是在 B 链上的 `receiveWormholeMessages` 实现中调用 `sendPayloadToEvm`。通常在这些场景中，你只有 A 链上的货币，但你仍然可以在 A 链上的交付请求中请求适当的金额作为你的 `receiverValue`。

你如何知道在 A 链上的交付中需要请求多少 receiver 值？不幸的是，你需要的金额取决于 B 链上的 delivery provider 可以提供的报价。我们的最佳建议是将这个 “receiverValue” 金额作为一个参数在你合约的端点上公开，并让应用程序的前端通过查询 B 链上的 WormholeRelayer 合约来确定要在此处传递的正确值。

HelloWormhole 仓库中包含了一个[示例合同](https://github.com/wormhole-foundation/hello-wormhole/blob/main/src/extensions/HelloWormholeConfirmation.sol)(和 [forge 测试](https://github.com/wormhole-foundation/hello-wormhole/blob/main/test/extensions/HelloWormholeConfirmation.t.sol)），使用上述建议，实现从 A 链到 B 链再到 C 链。

## 与其他 Wormhole 模块组合 - 请求传递现有的 Wormhole 信息

很多时候，我们希望传递一个已经发布过的 wormhole 信息（通过不同的合约）。

为此，请使用 [sendVaasToEvm](https://github.com/wormhole-foundation/wormhole/blob/main/ethereum/contracts/interfaces/relayer/IWormholeRelayer.sol#L149) 函数，该函数允许你指定额外已发布的 wormhole 信息，这些信息中相应的已签名的 VAAs 将作为调用 `targetAddress` 时的参数传递进来。

<pre class="language-solidity"><code class="lang-solidity">/**
 * @notice VaaKey identifies a wormhole message
 *
 * @custom:member chainId Wormhole chain ID of the chain where this VAA was emitted from
 * @custom:member emitterAddress Address of the emitter of the VAA, in Wormhole bytes32 format
 * @custom:member sequence Sequence number of the VAA
<strong> */
</strong>struct VaaKey {
    uint16 chainId;
    bytes32 emitterAddress;
    uint64 sequence;
}

function sendVaasToEvm(
        uint16 targetChain,
        address targetAddress,
        bytes memory payload,
        uint256 receiverValue,
        uint256 gasLimit,
        VaaKey[] memory vaaKeys,
        uint16 refundChain,
        address refundAddress
) external payable returns (uint64 sequence);
</code></pre>

有关此用法的示例，请参阅 Wormhole Solidity SDK 的 [sendTokenWithPayloadToEvm](https://github.com/wormhole-foundation/wormhole-solidity-sdk/blob/main/src/WormholeRelayerSDK.sol#L131) 实现，它使用 TokenBridge wormhole 模块发送 tokens！

{% hint style="info" %}
### Wormhole 集合完成了吗？

请告诉我们，这样我们就可以将你的项目列入我们的生态目录，并将你介绍给我们的全球多链社区！

[立即联系！](https://forms.clickup.com/45049775/f/1aytxf-10244/JKYWRUQ70AUI99F32Q)
{% endhint %}
