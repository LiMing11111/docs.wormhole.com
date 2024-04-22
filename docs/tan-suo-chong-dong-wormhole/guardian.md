# Guardians

## Guardian (守护者)

Wormhole 依赖一组分布式节点，这些节点监控几个区块链上的状态。在 Wormhole 中，这些节点被称为 Guardians。当前的 guardian 集合可以在 [Wormhole Explorer](https://wormhole.com/network/) 中看到。

Guardian 的角色是观察消息并签署相应的 payloads 每个 Guardian 都独立执行此步骤，然后将结果签名与其他 Guardians 的签名合并为最终步骤。由独立观察结果组成的多签名形成了一个多重签名，代表一个状态已被 wormhole 网络的大多数成员观察并同意。这些多重签名在 Wormhole 中被称为 VAAs。

## Guardian 网络

Guardian 网络旨在作为 Wormhole 的预言机组件，整个 Wormhole 生态系统建立在其技术基础上。它是 Wormhole 生态系统中最关键的元素，如果你想深入了解 Wormhole，这是最重要的组件。

为了不仅理解 Guardian 网络的工作方式，还要理解其工作原因，让我们首先回顾一下关键的设计考虑因素。为了成为一流的互操作平台，Wormhole 需要具备五个关键特性：

1. **去中心化** - 网络的控制需要在许多方之间分散。
2. **模块化** - 生态系统中的不同部分，如预言机、中继器、应用程序等，应保持尽可能独立和模块化，以便它们可以独立设计、修改和升级。
3. **链无关性** - Wormhole 应该能够支持不仅是EVM，还有像Solana、Algorand、Cosmos等链，甚至还包括尚未创建的平台。它也不应该让任何一个链成为单一的故障点。
4. **可扩展性** - Wormhole应能迅速并有效地保障平台上锁定或流通的大量资产的安全，并能够处理大量交易量。
5. **可升级性** - 随着去中心化计算生态系统的发展，Wormhole 需要能够改变其现有模块的实现，而不影响已经集成的项目方。

接下来，让我们逐一探讨 Wormhole 是如何实现这些目标的。

### 去中心化

去中心化是最大的关注点。以前的互操作解决方案大多完全集中，即使是使用诸如敌对中继器之类的新解决方案，仍然倾向于具有单点故障或低至 1 或 2 的勾结阈值。

设计去中心化预言机网络时，首先考虑的选项可能是 Proof-of-Stake (PoS) 系统——但这被证明是次优解。PoS 设计用于智能合约环境中的区块链共识，因此当一个网络需要同时验证多个区块链的数据输出而不仅仅是维护自己的智能合约时，它不太适合。尽管从去中心化的角度看起来很吸引人，但网络安全仍不清楚，它可能使实现其他一些目标变得更加困难。让我们探索其他选项。

下一个选项是直接冲向终点，使用零知识证明来保护网络。从去中心化的角度来看，这是一个好的解决方案，因为它实际上是无需信任的。然而，零知识证明仍是一个新兴技术，特别是在计算能力有限的链上验证，它们实际上是不可行的。这意味着需要一种多重签名形式来保护网络。

如果我们退一步看当前的 De-Fi 格局，大多数顶级区块链都由同一小部分验证器公司保护。目前，世界上有限的几家公司具备运营一流验证器公司的技能和资本。

如果协议能够将大量验证者公司联合起来，形成一个专门构建的共识机制，并针对链互操作性进行优化，那么这种设计可能比由代币经济学模型引导的网络更高效、更安全。假设验证者想要加入，Wormhole 实际可以容纳多少个验证者？

如果 Wormhole 使用阈值签名，答案基本上是“愿意参与的尽可能多”。然而，阈值签名在区块链世界中的支持不一，这意味着验证签名将是困难且昂贵的，最终限制了可扩展性和链无关性。因此，t-schnorr 多重签名呈现为最佳选项：便宜且支持良好，尽管其验证成本随签名数量线性增加。

综合考虑这些因素，19 似乎是最大数量并且是一个良好的折衷方案。如果需要 2/3 的签名达成共识，则需要在链上验证 13 个签名，从 gas 成本的角度来看这仍然是合理的。

与其通过代币经济学来保护网络，不如先让那些为整个 DeFi 的成功投入大量资金的强大公司参与进来，从而保护网络。这 19 位守护者并非匿名或小公司——他们中的许多都是加密货币领域中最大、最知名的验证者公司。当前的 Guardian 列表可以在[这里](https://wormhole.com/network/)查看。

这就是我们最终拥有 19 个 Guardian 的网络，每个 Guardian 拥有平等的权益，并加入到一个为权威证明共识机制中。随着阈值签名得到更好的支持，Guardian 集合可以扩大，一旦 ZKPs 普及，Guardian 网络将变得完全无需信任。

通过我们对去中心化的看法，其余的要素就都明了了。

### 模块化

Guardian 网络本身就很强大且值得信赖，因此不需要中继器等组件来为安全模型做出贡献。这使得 Wormhole 能够拥有简单的组件架构，这些组件在它们所做的事情上都非常出色且独立负责。这样，守护者只需要验证链上活动并生成 VAA，而中继器只需要与区块链交互并传递消息。

VAAs 的签名方案可以更改，而不影响下游用户，并且可以独立存在多种中继机制。xAssets 可以纯粹在应用层实现，跨链应用可以使用适合它们的任何组件。

### 链无关性

今天，Wormhole 支持的生态系统范围比任何其他互操作协议都广，因为它使用简单技术（t-schnorr 签名）、一个可适应的、异构的中继器模型和一个强大的验证器网络。

Wormhole 可以像为智能合约运行时开发核心合约一样快速扩展到新的生态系统。中继者不需要被纳入安全模型——他们只需要能够将消息上传到区块链。Guardians 能够观察每条链上的每笔交易，而无需走特殊的捷径。

### 可扩展性

Wormhole 的可扩展性表现良好，这一点从它在市场动荡时期也能处理巨大的总锁定价值（TVL）和交易量中可以看出。

运行 Guardian 的要求相对较高，因为他们需要为生态系统中的每个单独的区块链运行一个完整节点。这是有限数量的强大验证器公司对此设计有益的另一个原因。

然而，一旦所有的完整节点都在运行，Guardian 网络的实际计算和网络开销就会变得轻量。区块链本身的性能往往是 Wormhole 的瓶颈，而不会受限于 Guardian 网络内部。

### 可升级性

随着时间的推移，Guardian 集合可以通过使用阈值签名扩展到超过 19 个。将出现各种中继模型，每种都有其优点和缺点。ZKPs 可以在它们得到良好支持的链上使用。跨链应用生态系统将增长，跨链应用将与彼此日益交织。Wormhole 中的 API 非常少，大多数项目都是从集成商的角度来看实现细节。这为实现完全无信任的互操作性层创建了一条清晰的道路，该层涵盖了整个去中心化计算。