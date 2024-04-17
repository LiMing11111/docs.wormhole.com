# 开发跨链Dapps

如果你还没有阅读 Wormhole 的[介绍](../../)来了解什么是跨链开发以及开发者和 Dapps 如何从中受益，请从这里开始。

[VAAs](../../tan-suo-chong-dong-wormhole/vaa.md) 是 Wormhole 协议中的核心信息传递原型。你可以将其视为跨链数据包，每当跨链应用合约与核心合约进行交互时，都会发出这些数据包。

在 Wormhole 的环境下，[中继器](../../tan-suo-chong-dong-wormhole/relayer.md)是将 Verifiable Action Approvals（VAAs）发送到其目标链的进程，在 Wormhole 的安全模型中扮演着至关重要的作用。它们无法破坏安全性，只会影响有效性，并充当 VAAs 的交付机制，但没有能力篡改结果。

在构建跨链应用程序时，使用 Wormhole 传递消息（VAAs）的方式主要有两种。

1. [自动中继](./#zi-dong-zhong-ji) - 无需链下代码
2. [专业中继](./#zhuan-ye-zhong-ji-qi) - 可能需要一些链下代码

{% hint style="info" %}
蓝色框出的组件是开发人员必须实现的组件。The components outlined in **blue** are those that must be implemented by the developer
{% endhint %}

### 自动中继

{% hint style="warning" %}
目前只有 EVM 环境支持自动中继。
{% endhint %}

![标准中继器](../../.gitbook/assets/auto-relayer.png)

有了自动中继功能，只需制定合同。将信息传递交给服务提供商即可。

[阅读更多](standard-relayer.md)

[快速开始](../tutorials/hello-wormhole/)

### 专业中继器

![专业中继器](../../.gitbook/assets/specialized-relayer.png)

通过专业中继，开发者可以与 [Wormhole 支持的任何区块链](../../blockchain-environments/)进行通信，并可自由选择交付策略。

[阅读更多](specialized-relayer.md)

[快速开始](../tutorials/relayer/)

### 更多

更多教程请点击[这里](../tutorials/)。
