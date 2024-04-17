# 安全性

### 核心安全假设

Wormhole 的核心是由验证和签署消息的守护者节点网络提供保护。如果绝大多数（例如 19 名中的 13 名）守护者节点签署了同一条消息，则可以认为该消息有效。目标链上的智能合约将在批准任何交易之前验证消息的签名和格式。

* Wormhole 的核心安全原语是其签名消息（已签名的 VAA）。
* Guardian 网络目前由 19 家全球顶级验证器公司提供保护，[点击此处查看列表](https://wormhole.com/network/)。
* 当核心合约集成方提出请求时，守护者节点会生成签名的状态证明（已签名的 VAA）。
* 每个守护者节点都运行 Wormhole 网络中每个区块链的全节点（而不是轻节点）。这意味着，如果区块链遭受共识攻击或硬分叉，区块链将与网络断开连接，而不是生成无效的签名 VAA。
* 任何已签名的 VAA 都可以由任何其他链的核心合约验证其真实性。
* 中继器在 Wormhole 生态系统中被认为是不可信的。

> 总结:
>
> * 核心集成商不会面临来自他们未集成的链和合约的风险。
> * 默认情况下，您只信任 Wormhole 的签名过程和您所在链的核心合约。
> * 您可以根据需要扩展合约和链的依赖关系。

除了核心安全假设之外，还有许多其他因素影响去中心化平台的实际安全性。以下是为确保 Wormhole 平台安全而采取的其他措施的更多信息。

### 守护者网络（Guardian Network）

Wormhole 是一个不断发展的平台。虽然 Guardian 集（即验证者集）目前包含 19 个验证者，但这主要是受到当前区块链技术的限制。

#### **治理**

治理是合约升级的过程。守护者节点手动对源自守护者网络内部的治理提案进行投票，然后提交给生态系统合约。

这意味着治理操作与系统的其他部分遵循相同的安全标准。任何治理行动都需要 2/3 绝对多数的守护者通过。

治理消息可以针对任何 Wormhole 模块，包括核心合约以及所有当前部署的代币桥合约。当守护者签署此类消息时，其签名意味着对相关操作进行了投票。一旦超过 2/3 的守护者进行了签名，该消息和治理行为即被视为有效。

所有治理行为和合约升级均通过 Wormhole 的链上治理系统进行管理。

通过治理，守护者能够：

* 更改当前的守护者集（即验证者集）
* 扩展守护者集
* 升级生态系统合约实施

治理系统在核心存储库中完全开源。请[参阅本节](https://docs.wormhole.com/wormhole/explore-wormhole/security#open-source)了解合约细节。

### 监控

Wormhole 纵深防御策略的另一个关键要素是，每个守护者都是一家能力强且经验丰富的验证公司，他们拥有一套完善的流程和机制来运行、监控和保护区块链操作。这种异构监控方法增加了检测到欺诈活动的可能性并减少了系统中单点故障的发生。

守护者节点不仅运行 Wormhole 验证器，还为 Wormhole 内部每个区块链运行验证器，这使他们能够在去中心化计算中执行整体监控，而不仅仅是在几个单一的点上。

守护者监控的内容包括：

* 每个区块链的区块生成和共识。如果某个区块链的共识遭到破坏，它就会从网络中断开，直到守护者节点解决问题。
* 智能合约级别数据。通过 Governor 等进程不断监控所有支持的区块链上的流通供应和代币变动。
* 守护者级别的活动。守护者网络作为一个自治的去中心化计算网络，拥有自己的区块链（Gateway）。

### Gateway 和资产层保护

Wormhole 生态系统最强大的方面之一是，守护者实际上可以使用完整状态的 DeFi。

Gateway 是一种基于 Cosmos 的区块链，在 Guardian 网络内部运行，因此 Guardians 可以根据所有区块链（而不仅仅是一个区块链）的当前状态有效执行智能合约。

除了核心假设之外，这还可以为 Wormhole 资产层提供额外的保护：

* **全局会计系统（Global Accountant）**：Global Accountant 会追踪所有链上Wormhole 资产的总流通供应量，并防止任何区块链连接违反供应不变性的资产。

除 Global Accountant 外，守护者只能签署不违反 [Governor](https://github.com/wormhole-foundation/wormhole/blob/main/whitepapers/0007\_governor.md) 要求的转账。Governor 跟踪所有区块链的流入和流出，并延迟传输可能存在漏洞的的可疑转移。

### 开源

Wormhole 是开放构建的，并且始终是开源的。

* [Wormhole Core Repository](https://github.com/wormhole-foundation/wormhole)
* [Wormhole Foundation Github Organization](https://github.com/wormhole-foundation)
* [Wormhole Contract Deployments](core-contracts.md)

### 审计

Wormhole 接受了大量审计，已完成 16 项第三方安全审计，总共启动了 25+ 项第三方审计。

Wormhole 已接受以下公司的安全审计，并将继续寻求更多审计：

* Trail of Bits
* Neodyme
* Kudelski
* OtterSec
* Certik
* Hacken
* Zellic
* Coinspect
* Halborn

最新的审计列表以及最终报告可以在[此处](https://github.com/wormhole-foundation/wormhole/blob/main/SECURITY.md#3rd-party-security-audits)找到。

### 漏洞赏金（Bug Bounties）

Wormhole 拥有软件开发领域最大的错误赏金计划之一，并多次表现出与白帽社区合作的意愿/承诺。

Wormhole 创建的两个漏洞赏金计划：

* 一个 [Immunefi](https://immunefi.com/bounty/wormhole/) 计划,
* 以及一个[自托管计划（self-hosted program）](https://wormhole.com/bounty/)

两个平台的最高金额均为 250 万美元。

如果您有兴趣为 Wormhole 安全做出贡献，请参阅本部分的“[白帽指南](https://github.com/wormhole-foundation/wormhole/blob/main/SECURITY.md#white-hat-hacking)”部分，并确保遵循 [Wormhole 贡献者指南](https://github.com/wormhole-foundation/wormhole/blob/main/CONTRIBUTING.md)。

有关提交错误赏金计划的更多信息，请[参阅此处](https://wormhole.com/bounty/)。

### 了解更多

* 官方代码仓库中的 [SECURITY.md](https://github.com/wormhole-foundation/wormhole/blob/main/SECURITY.md) 具有最新的安全策略和更新。
