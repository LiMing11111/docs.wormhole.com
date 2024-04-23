# Hello Wormhole

本教程包含一个 solidity 合约（`HelloWormhole.sol`），可部署到多个 EVM 链上，形成一个功能完整的跨链应用程序。

具体来说，我们将编写一个合约并将其部署到多个链上，允许用户从一个合约中请求，在不同链上的另一个合约中发出 `GreetingReceived` 事件。

用户还可以通过这种方式为自己的定制问候语支付费用，以便在没有任何 gas 费的链上发出问候语！

## 入门

[源码仓库](https://github.com/wormhole-foundation/hello-wormhole)中包含以下内容：

* Solidity 代码示例
* Forge 本地测试配置示例
* 测试网部署脚本
* 测试网测试配置示例

### 环境配置

* Node 16.14.1 或更高版本， npm 8.5.0 或更高版本： [https://docs.npmjs.com/downloading-and-installing-node-js-and-npm](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm)
* forge 0.2.0 或更高版本：[https://book.getfoundry.sh/getting-started/installation](https://book.getfoundry.sh/getting-started/installation)

### 本地测试

从 github 上下载代码并进入到其目录下，然后构建并测试。

```bash
git clone https://github.com/wormhole-foundation/hello-wormhole.git
cd hello-wormhole
npm run build
forge test
```

预期输出为

```bash
Running 1 test for test/HelloWormhole.t.sol:HelloWormholeTest
[PASS] testGreeting() (gas: 777229)
Test result: ok. 1 passed; 0 failed; finished in 3.98s
```

### 部署到测试网

你将需要一个至少有 0.05 Testnet AVAX 和 0.01 Testnet CELO 的钱包。

* [在此获取 testnet AVAX](https://core.app/tools/testnet-faucet/?token=C)
* [在此获取 testnet CELO](https://faucet.celo.org/alfajores)

```bash
EVM_PRIVATE_KEY=your_wallet_private_key npm run deploy
```

### 在测试网上测试

你将需要一个至少有 0.02 Testnet AVAX 的钱包。[在此获取 testnet AVAX](https://core.app/tools/testnet-faucet/?token=C)

你还必须将合约部署到测试网络上（如上文所述）。

为了测试在测试网上发送和接收信息，请按如下方式执行测试：

```bash
EVM_PRIVATE_KEY=your_wallet_private_key npm run test
```

## HelloWormhole 跨链合约说明

让我们以一个简单的 HelloWorld solidity 应用程序为例，并将其实现跨链！

### 单链 HelloWorld solidity 合约

此单链 HelloWorld 智能合约允许用户发送问候语。换句话说，它允许用户在发送问候语时触发 `GreetingReceived` 事件！

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract HelloWorld {
    event GreetingReceived(string greeting, address sender);

    string[] public greetings;

    /**
     * @notice Returns the cost (in wei) of a greeting
     */
    function quoteGreeting() public view returns (uint256 cost) {
        return 0;
    }

    /**
     * @notice Updates the list of 'greetings'
     * and emits a 'GreetingReceived' event with 'greeting'
     */
    function sendGreeting(
        string memory greeting
    ) public payable {
        uint256 cost = quoteGreeting();
        require(msg.value == cost);
        emit GreetingReceived(greeting, msg.sender);
        greetings.push(greeting);
    }
}
```

### 使用 Wormhole Automatic Relayers实现 HelloWorld 跨链传输

假设我们希望用户能够通过他们的 Ethereum 钱包请求向 Avalanche 发送问候，反之亦然。

让我们开始编写一个合约，将其部署到 Ethereum、Avalanche或其他任何链上，使每个合约之间都能自由发送问候，而不受链的限制。

我们需要执行以下函数：

```solidity
    /**
     * @notice Updates the list of 'greetings'
     * and emits a 'GreetingReceived' event with 'greeting'
     * on the HelloWormhole contract at
     * chain 'targetChain' and address 'targetAddress'
     */
    function sendCrossChainGreeting(
        uint16 targetChain,
        address targetAddress,
        string memory greeting
    ) public payable;
```

Wormhole Relayer 合约正是让我们做到了这一点！让我们来看一下 Wormhole Relayer 合约的界面。

```solidity
    /**
     * @notice Publishes an instruction for the default delivery provider
     * to relay a payload to the address `targetAddress` on chain `targetChain`
     * with gas limit `gasLimit` and `msg.value` equal to `receiverValue`
     *
     * `targetAddress` must implement the IWormholeReceiver interface
     *
     * This function must be called with `msg.value` equal to `quoteEVMDeliveryPrice(targetChain, receiverValue, gasLimit)`
     *
     * Any refunds (from leftover gas) will be paid to the delivery provider. In order to receive the refunds, use the `sendPayloadToEvm` function
     * with `refundChain` and `refundAddress` as parameters
     *
     * @param targetChain in Wormhole Chain ID format
     * @param targetAddress address to call on targetChain (that implements IWormholeReceiver)
     * @param payload arbitrary bytes to pass in as parameter in call to `targetAddress`
     * @param receiverValue msg.value that delivery provider should pass in for call to `targetAddress` (in targetChain currency units)
     * @param gasLimit gas limit with which to call `targetAddress`.
     * @return sequence sequence number of published VAA containing delivery instructions
     */
    function sendPayloadToEvm(
        uint16 targetChain,
        address targetAddress,
        bytes memory payload,
        uint256 receiverValue,
        uint256 gasLimit
    ) external payable returns (uint64 sequence);
```

Wormhole Relayer network 由 **Delivery Providers 提供支持，**他们负责监 Wormhole Relayer 的交付请求，并按照指示将其传送到目标链。

作为交换，你需要在 `targetChain` 上调用 `targetAddress` 上的合约并支付你的合约所消耗的gas 费，他们会收取源链费用。所收取的费用将取决于目标网络的条件，并且可以向交付提供者请求该费用：

```
(deliveryPrice,) = quoteEVMDeliveryPrice(targetChain, receiverValue, gasLimit)
```

因此，按照此接口，我们可以通过简单地调用 sendPayloadToEvm 来实现 `sendCrossChainGreeting` ，其中的 payload 是我们想要发送的一些信息，例如问候语和问候语的发送者。

```solidity
    uint256 constant GAS_LIMIT = 50_000;

    IWormholeRelayer public immutable wormholeRelayer;

    /**
     * @notice Returns the cost (in wei) of a greeting
     */
    function quoteCrossChainGreeting(
        uint16 targetChain
    ) public view returns (uint256 cost) {
        // Cost of requesting a message to be sent to
        // chain 'targetChain' with a gasLimit of 'GAS_LIMIT'
        (cost, ) = wormholeRelayer.quoteEVMDeliveryPrice(
            targetChain,
            0,
            GAS_LIMIT
        );
    }

    /**
     * @notice Updates the list of 'greetings'
     * and emits a 'GreetingReceived' event with 'greeting'
     * on the HelloWormhole contract at
     * chain 'targetChain' and address 'targetAddress'
     */
    function sendCrossChainGreeting(
        uint16 targetChain,
        address targetAddress,
        string memory greeting
    ) public payable {
        bytes memory payload = abi.encode(greeting, msg.sender);
        uint256 cost = quoteCrossChainGreeting(targetChain);
	    require(msg.value == cost, "Incorrect payment");
        wormholeRelayer.sendPayloadToEvm{value: cost}(
            targetChain,
            targetAddress,
            payload,
            0, // no receiver value needed
            GAS_LIMIT
        );
    }

```

不过，该系统的一个关键部分是 `targetAddress` 的合约必须实现 `IWormholeReceiver` 接口。

由于我们希望通过 `HelloWormhole` 合约发送和接收信息，因此我们必须实现此接口。

```solidity
// SPDX-License-Identifier: Apache 2

pragma solidity ^0.8.0;

/**
 * @notice Interface for a contract which can receive Wormhole messages.
 */
interface IWormholeReceiver {
    /**
     * @notice When a `send` is performed with this contract as the target, this function will be
     *     invoked by the WormholeRelayer contract
     *
     * NOTE: This function should be restricted such that only the Wormhole Relayer contract can call it.
     *
     * We also recommend that this function checks that `sourceChain` and `sourceAddress` are indeed who
     *       you expect to have requested the calling of `send` on the source chain
     *
     * The invocation of this function corresponding to the `send` request will have msg.value equal
     *   to the receiverValue specified in the send request.
     *
     * If the invocation of this function reverts or exceeds the gas limit
     *   specified by the send requester, this delivery will result in a `ReceiverFailure`.
     *
     * @param payload - an arbitrary message which was included in the delivery by the
     *     requester.
     * @param additionalVaas - Additional VAAs which were requested to be included in this delivery.
     *   They are guaranteed to all be included and in the same order as was specified in the
     *     delivery request.
     * @param sourceAddress - the (wormhole format) address on the sending chain which requested
     *     this delivery.
     * @param sourceChain - the wormhole chain ID where this delivery was requested.
     * @param deliveryHash - the VAA hash of the deliveryVAA.
     *
     * NOTE: These signedVaas are NOT verified by the Wormhole core contract prior to being provided
     *     to this call. Always make sure `parseAndVerify()` is called on the Wormhole core contract
     *     before trusting the content of a raw VAA, otherwise the VAA may be invalid or malicious.
     */
    function receiveWormholeMessages(
        bytes memory payload,
        bytes[] memory additionalVaas,
        bytes32 sourceAddress,
        uint16 sourceChain,
        bytes32 deliveryHash
    ) external payable;
}
```

在源链上调用 `sendPayloadToEvm` 方法后，链下的 Delivery Provider 将获取与消息对应的 VAA。然后，它将在指定的 `targetChain` 和 `targetAddress` 上调用 `receiveWormholeMessages` 方法。

因此，在 receiveWormholeMessages 中，我们要：

1. 更新最新的问候语
2. 发出一个 'GreetingReceived' 事件，其中包含 'greeting' 和问候的发送者

> 注意：只有 Wormhole Relayer 合约才能调用 receiveWormholeMessages，这一点至关重要。

为了确保 payload 的有效性，我们必须将此函数的 msg.sender 限制为仅为 Wormhole Relayer 合约。否则，任何人都可以使用虚假问候语、源链和源发送者调用此 receiveWormholeMessages 端点。

这样，我们就有了一个完整的合约，它可以部署到许多 EVM 链上，并在 Wormhole 的支持下形成一个完整的跨链应用！

拥有任何钱包的用户都可以请求在系统中的任何一条链上发出问候。

### 它是如何工作的？

[查看第 2 部分](hello-wormhole-explained.md)深入了解Wormhole Relayer 如何使用适当的输入调用其他区块链上的合约！

### 完整的跨链 HelloWormhole solidity 合约

查看 [HelloWormhole.sol 合约的完整实现](https://github.com/wormhole-foundation/hello-wormhole/blob/main/src/HelloWormhole.sol)以及[包含测试基础框架的完整 Github 仓库](https://github.com/wormhole-foundation/hello-wormhole/)

{% hint style="info" %}
### Wormhole 集成完成了吗？

请告诉我们，这样我们就可以将你的项目列入我们的生态目录，并将你介绍给我们的全球多链社区！

[立即联系！](https://forms.clickup.com/45049775/f/1aytxf-10244/JKYWRUQ70AUI99F32Q)
{% endhint %}
