# CCTP

Wormhole Connect 及其 SDK 支持在 Circle 的跨链转账协议支持的所有网络之间快速、低成本的进行原生 USDC 的桥接。[跨链转账协议](https://www.circle.com/en/cross-chain-transfer-protocol)是 Circle 的原生 USDC 跨链转移认证服务。

虽然这个协议与 Wormhole 本身完全独立，但 Wormhole 基于 CCTP 构建，并增加了几项有用的增强功能，使其更易于使用且对终端用户更有用。这些功能包括自动中继（这样用户就不需要自己兑换 USDC 转账），目的链上的 gas 费支付（这样用户可以在目的链上转移 USDC 而不需要支付 gas 费），以及 gas 补给（允许用户在成功转账后将一些 USDC 转换为目的地的 gas 代币）。

{% hint style="info" %}
Wormhole 支持所有 CCTP 支持的链，但目前只有[少数几条链](https://developers.circle.com/stablecoins/docs/supported-domains)得到了 Circle 的支持。
{% endhint %}

您可以通过嵌入 [Connect Widget](../../wh-connect.md) 或直接整合 Connect [SDK](sdk.md) 来使用 Wormhole 的 CCTP 驱动的 USDC 桥接。

### 自动中继

In order to complete a CCTP transfer, the [Circle Attestation](https://developers.circle.com/stablecoins/reference/getattestation) must be delivered to the destination chain.

为了完成 CCTP 转账，必须将 [Circle Attestation](https://developers.circle.com/stablecoins/reference/getattestation) 送达目的链。 在某些情况下，认证的传递可能是困难或不可能的。例如，在浏览器环境中，当用户不希望等待最终确认来传递认证时。为了解决这个困难，可以使用 Wormhole CCTP 中继器，要么通过 Wormhole [Connect Widget](../../wh-connect.md)，要么更直接地通过 [Connect SDK](../../../reference/sdk-docs/)。&#x20;

| Chain           | Fee      |
| --------------- | -------- |
| Ethereum        | 1.0 USDC |
| Everything else | 0.1 USDC |

使用自动中继功能的另一个优势是有机会在目的链上向接收者转移一些原生 gas。这个功能被称为`native gas dropoff`。

能够执行原生 gas 补给解决了一个常见问题，即用户可能持有 USDC 余额，但没有原生 gas 来执行后续交易。

{% hint style="info" %}
原生 gas 补给的限制是待定的。
{% endhint %}

### 通过 Connect Widget 的 CCTP

使用 Wormhole Connect 小部件支持 CCTP USDC 转账非常简单。在您的去中心化应用中直接启用 USDC 桥接，只需几行代码，即可获得 Wormhole 构建的所有用户友好功能。&#x20;

相关介绍可以在[这里](https://app.gitbook.com/o/SK9htrrKEoxzXNcFitC5/s/uLLPBOibnlYmSJX40vn0/)查看。

### 通过 Connect SDK 的 CCTP

使用 Connect SDK 支持手动或自动的 CCTP 转账都非常直接。&#x20;

完整教程可在[这里](https://app.gitbook.com/o/SK9htrrKEoxzXNcFitC5/s/q6sqV4ZI7t3HYi0Fucx5/\~/changes/30/kuai-su-kai-shi/tutorials/cctp/sdk%E3%80%81)查看。
