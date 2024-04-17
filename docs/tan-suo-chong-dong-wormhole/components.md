# 架构

## 架构

Wormhole 由几个关键组件组成。在我们深入探讨每个组件之前，让我们先了解一下主要部件的名称及其如何协同工作。

![详细的工作机制](../.gitbook/assets/core-concepts/detailed-flow.jpg)

### 链上组件

* 发射器（Emitter） - 一个调用核心合约（Core Contract）上发布消息方法的合约。核心合约将在交易日志（Transaction Logs）中写入一个事件，包括关于发射器（Emitter）和序列号（Sequence Number）的详细信息，以识别消息。这可能是您的 xDapp 或现有的生态系统协议。
* Wormhole 核心合约（Wormhole Core Contract） - 主合约，这是 Guardian 遵守的合约，从根本上实现了跨链通信。
* 交易日志（Transaction Logs）- 特定于区块链的日志，使 Guardian 能够观察核心合约发出的消息。

## 链下组件

* 守护者网络（Guardian Network） - 存在于自己的 P2P 网络中的验证者。守护者观察并验证核心合约在每个支持的链上发出的消息，以生成 VAAs（已签名的消息）。
* 守护者（Guardian） - 守护者网络中为 VAA 多重签名做出贡献的 19 个验证者之一。
* Spy - 一个订阅守护者网络内发布消息的守护进程。Spy 可以观察和转发网络流量，这有助于扩大 VAA 分发范围。
* API - 一个 REST 服务器，用于检索 VAA 或守护者网络的详细信息。
* VAAs - 可验证的行动批准（Verifiable Action Approvals, VAAs）是对 Wormhole 核心合约观察到的消息的签名证明。
* 中继器（Relayer） - 将 VAA 中继到目标链的任何链下进程。
  * 标准中继器（Standard Relayers） - 一个去中心化的中继器网络，通过 Wormhole 中继合约在链上请求传递消息。也称为通用中继器（Generic Relayers）。
  * 专用中继器（Specialized Relayers） - 仅处理特定协议或跨链应用的 VAAs 的中继器。它们可以在链下执行自定义逻辑，从而减少 Gas 成本并提高跨链兼容性。目前，跨链应用开发者负责开发和托管专用中继器。
