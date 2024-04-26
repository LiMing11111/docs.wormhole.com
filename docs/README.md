# 介绍

Wormhole 是一种通用的**信息传递协议**，可实现区块链之间的通信。

![Overview](.gitbook/assets/oversimplified.jpg)

{% hint style="success" %}
以上是对该协议的一个_过于简化的_说明，有关架构和组件的详细信息，请参阅[此处](tan-suo-chong-dong-wormhole/components.md)
{% endhint %}

这种简单的**信息传递协议**使开发者和由开发者构建的[跨链应用程序](reference/glossary.md#xdapps)的用户能够充分利用多个生态系统的优势。

### 什么不是 Wormhole？

Wormhole 本身并_不是_一个区块链，它提供了一种区块链或 rollups 之间的通信方式。

Wormhole _不是一个_ token bridge，尽管[基于 Wormhole 的协议](https://www.portalbridge.com/#/transfer)可以实现这一目的。

### Wormhole 可以用来做什么？

考虑以下几个现在使用 Wormhole 可以实现的潜在应用示例：

1. **跨链交易**：使用 [Wormhole Connect](kuai-su-kai-shi/wh-connect.md)，开发者可以建立一个允许从任何 Wormhole 连接链进行存款的交易所，从而大幅增加用户可以访问的流动性。
2. **跨链治理**：如果不同网络上的一组 NFT 收藏品希望它们的持有者就一项联合提案进行投票，他们可以选择一条“投票”的链，并使用 Wormhole 将他们在不同链上投的票传达给投票链。
3. **跨链游戏**： 可以在像 Solana 这样的高性能网络上构建并玩游戏，并且它的奖励作为 NFT 的形式在不同的网络上发行，例如以太坊。

## 开始

### 快速入门教程

我们提供了教程以便快速入门，并解释了涉及的概念。

<table data-card-size="large" data-view="cards" data-full-width="false"><thead><tr><th></th><th></th><th data-hidden data-card-target data-type="content-ref"></th><th data-hidden data-card-cover data-type="files"></th></tr></thead><tbody><tr><td>快速入门 - 链下</td><td>将 Wormhole Connect 集成到新的或现有的 web UI 中</td><td><a href="tutorials/quick-start/wormhole-connect/wh-connect.md">wh-connect.md</a></td><td><a href=".gitbook/assets/wh-connect-default.png">wh-connect-default.png</a></td></tr><tr><td>快速入门 - 链上</td><td>发送你的第一条跨链消息</td><td><a href="kuai-su-kai-shi/cross-chain-dev/">cross-chain-dev</a></td><td><a href=".gitbook/assets/wh-line-art.png">wh-line-art.png</a></td></tr></tbody></table>

更多教程请点击[此处](tutorials/quick-start/)。

### 探索

了解更多关于 Wormhole 生态系统、组件和协议的信息。

<table data-card-size="large" data-view="cards" data-full-width="false"><thead><tr><th></th><th></th><th data-hidden data-card-target data-type="content-ref"></th><th data-hidden data-card-cover data-type="files"></th></tr></thead><tbody><tr><td><strong>结构</strong></td><td>深入了解协议的组成部分</td><td><a href="tan-suo-chong-dong-wormhole/components.md">components.md</a></td><td><a href=".gitbook/assets/detailed-flow.jpg">detailed-flow.jpg</a></td></tr><tr><td><strong>协议规范</strong></td><td>进一步了解基于 Wormhole 的协议</td><td><a href="https://github.com/wormhole-foundation/wormhole/tree/main/whitepapers">https://github.com/wormhole-foundation/wormhole/tree/main/whitepapers</a></td><td><a href=".gitbook/assets/protocols.png">protocols.png</a></td></tr></tbody></table>

### Demos

Demos 提供了比教程更真实的实现方式。

<table data-card-size="large" data-view="cards" data-full-width="false"><thead><tr><th></th><th></th><th data-hidden data-card-target data-type="content-ref"></th><th data-hidden data-card-cover data-type="files"></th></tr></thead><tbody><tr><td><strong>Wormhole Scaffolding</strong></td><td>使用 Scaffolding 代码库快速启动项目</td><td><a href="https://github.com/wormhole-foundation/wormhole-scaffolding">https://github.com/wormhole-foundation/wormhole-scaffolding</a></td><td><a href=".gitbook/assets/scaffolding.jpg">scaffolding.jpg</a></td></tr><tr><td><strong>xDapp book 项目</strong></td><td>运行并学习示例程序</td><td><a href="https://github.com/wormhole-foundation/xdapp-book/tree/main/projects">https://github.com/wormhole-foundation/xdapp-book/tree/main/projects</a></td><td><a href=".gitbook/assets/projects.png">projects.png</a></td></tr></tbody></table>

更多 Demos 请点击此处[此处](kuai-su-kai-shi/demos.md)。

{% hint style="info" %}
### Wormhole 集成完成了吗？

请告诉我们，这样我们就可以将你的项目列入我们的生态目录，并将你介绍给我们的全球多链社区！

[立即联系！](https://forms.clickup.com/45049775/f/1aytxf-10244/JKYWRUQ70AUI99F32Q)
{% endhint %}

## 支持的区块链

Wormhole 支持越来越多的区块链

<table data-view="cards" data-full-width="false"><thead><tr><th></th><th data-hidden data-card-target data-type="content-ref"></th><th data-hidden data-card-cover data-type="files"></th></tr></thead><tbody><tr><td><strong>Acala</strong></td><td><a href="blockchain-environments/evm/#acala">#acala</a></td><td><a href=".gitbook/assets/chain-icons/acala.svg">acala.svg</a></td></tr><tr><td><strong>Algorand</strong></td><td><a href="blockchain-environments/algorand/#algorand">#algorand</a></td><td><a href=".gitbook/assets/chain-icons/algorand.svg">algorand.svg</a></td></tr><tr><td><strong>Aptos</strong></td><td><a href="blockchain-environments/aptos/#aptos">#aptos</a></td><td><a href=".gitbook/assets/chain-icons/aptos.svg">aptos.svg</a></td></tr><tr><td><strong>Arbitrum</strong></td><td><a href="blockchain-environments/evm/#arbitrum">#arbitrum</a></td><td><a href=".gitbook/assets/chain-icons/arbitrum.svg">arbitrum.svg</a></td></tr><tr><td><strong>Arbitrum Sepolia</strong></td><td><a href="blockchain-environments/evm/#arbitrum_sepolia">#arbitrum_sepolia</a></td><td><a href=".gitbook/assets/chain-icons/arbitrum_sepolia.svg">arbitrum_sepolia.svg</a></td></tr><tr><td><strong>Aurora</strong></td><td><a href="blockchain-environments/evm/#aurora">#aurora</a></td><td><a href=".gitbook/assets/chain-icons/aurora.svg">aurora.svg</a></td></tr><tr><td><strong>Avalanche</strong></td><td><a href="blockchain-environments/evm/#avalanche">#avalanche</a></td><td><a href=".gitbook/assets/chain-icons/avalanche.svg">avalanche.svg</a></td></tr><tr><td><strong>Base</strong></td><td><a href="blockchain-environments/evm/#base">#base</a></td><td><a href=".gitbook/assets/chain-icons/base.svg">base.svg</a></td></tr><tr><td><strong>Base Sepolia</strong></td><td><a href="blockchain-environments/evm/#base_sepolia">#base_sepolia</a></td><td><a href=".gitbook/assets/chain-icons/base_sepolia.svg">base_sepolia.svg</a></td></tr><tr><td><strong>BNB Smart Chain</strong></td><td><a href="blockchain-environments/evm/#bsc">#bsc</a></td><td><a href=".gitbook/assets/chain-icons/bsc.svg">bsc.svg</a></td></tr><tr><td><strong>Celestia</strong></td><td><a href="blockchain-environments/cosmwasm/#celestia">#celestia</a></td><td><a href=".gitbook/assets/chain-icons/celestia.svg">celestia.svg</a></td></tr><tr><td><strong>Celo</strong></td><td><a href="blockchain-environments/evm/#celo">#celo</a></td><td><a href=".gitbook/assets/chain-icons/celo.svg">celo.svg</a></td></tr><tr><td><strong>Cosmoshub</strong></td><td><a href="blockchain-environments/cosmwasm/#cosmoshub">#cosmoshub</a></td><td><a href=".gitbook/assets/chain-icons/cosmoshub.svg">cosmoshub.svg</a></td></tr><tr><td><strong>Dymension</strong></td><td><a href="blockchain-environments/cosmwasm/#dymension">#dymension</a></td><td><a href=".gitbook/assets/chain-icons/dymension.svg">dymension.svg</a></td></tr><tr><td><strong>Ethereum</strong></td><td><a href="blockchain-environments/evm/#ethereum">#ethereum</a></td><td><a href=".gitbook/assets/chain-icons/ethereum.svg">ethereum.svg</a></td></tr><tr><td><strong>Evmos</strong></td><td><a href="blockchain-environments/cosmwasm/#evmos">#evmos</a></td><td><a href=".gitbook/assets/chain-icons/evmos.svg">evmos.svg</a></td></tr><tr><td><strong>Fantom</strong></td><td><a href="blockchain-environments/evm/#fantom">#fantom</a></td><td><a href=".gitbook/assets/chain-icons/fantom.svg">fantom.svg</a></td></tr><tr><td><strong>Gnosis</strong></td><td><a href="blockchain-environments/evm/#gnosis">#gnosis</a></td><td><a href=".gitbook/assets/chain-icons/gnosis.svg">gnosis.svg</a></td></tr><tr><td><strong>Ethereum Holesky</strong></td><td><a href="blockchain-environments/evm/#holesky">#holesky</a></td><td><a href=".gitbook/assets/chain-icons/holesky.svg">holesky.svg</a></td></tr><tr><td><strong>Injective</strong></td><td><a href="blockchain-environments/cosmwasm/#injective">#injective</a></td><td><a href=".gitbook/assets/chain-icons/injective.svg">injective.svg</a></td></tr><tr><td><strong>Karura</strong></td><td><a href="blockchain-environments/evm/#karura">#karura</a></td><td><a href=".gitbook/assets/chain-icons/karura.svg">karura.svg</a></td></tr><tr><td><strong>Klaytn</strong></td><td><a href="blockchain-environments/evm/#klaytn">#klaytn</a></td><td><a href=".gitbook/assets/chain-icons/klaytn.svg">klaytn.svg</a></td></tr><tr><td><strong>Kujira</strong></td><td><a href="blockchain-environments/cosmwasm/#kujira">#kujira</a></td><td><a href=".gitbook/assets/chain-icons/kujira.svg">kujira.svg</a></td></tr><tr><td><strong>Mantle</strong></td><td><a href="blockchain-environments/evm/#mantle">#mantle</a></td><td><a href=".gitbook/assets/chain-icons/mantle.svg">mantle.svg</a></td></tr><tr><td><strong>Moonbeam</strong></td><td><a href="blockchain-environments/evm/#moonbeam">#moonbeam</a></td><td><a href=".gitbook/assets/chain-icons/moonbeam.svg">moonbeam.svg</a></td></tr><tr><td><strong>NEAR</strong></td><td><a href="blockchain-environments/near/#near">#near</a></td><td><a href=".gitbook/assets/chain-icons/near.svg">near.svg</a></td></tr><tr><td><strong>Neon</strong></td><td><a href="blockchain-environments/evm/#neon">#neon</a></td><td><a href=".gitbook/assets/chain-icons/neon.svg">neon.svg</a></td></tr><tr><td><strong>Neutron</strong></td><td><a href="blockchain-environments/cosmwasm/#neutron">#neutron</a></td><td><a href=".gitbook/assets/chain-icons/neutron.svg">neutron.svg</a></td></tr><tr><td><strong>Oasis</strong></td><td><a href="blockchain-environments/evm/#oasis">#oasis</a></td><td><a href=".gitbook/assets/chain-icons/oasis.svg">oasis.svg</a></td></tr><tr><td><strong>Optimism</strong></td><td><a href="blockchain-environments/evm/#optimism">#optimism</a></td><td><a href=".gitbook/assets/chain-icons/optimism.svg">optimism.svg</a></td></tr><tr><td><strong>Optimism Sepolia</strong></td><td><a href="blockchain-environments/evm/#optimism_sepolia">#optimism_sepolia</a></td><td><a href=".gitbook/assets/chain-icons/optimism_sepolia.svg">optimism_sepolia.svg</a></td></tr><tr><td><strong>Osmosis</strong></td><td><a href="blockchain-environments/cosmwasm/#osmosis">#osmosis</a></td><td><a href=".gitbook/assets/chain-icons/osmosis.svg">osmosis.svg</a></td></tr><tr><td><strong>Polygon</strong></td><td><a href="blockchain-environments/evm/#polygon">#polygon</a></td><td><a href=".gitbook/assets/chain-icons/polygon.svg">polygon.svg</a></td></tr><tr><td><strong>Polygon Sepolia</strong></td><td><a href="blockchain-environments/evm/#polygon_sepolia">#polygon_sepolia</a></td><td></td></tr><tr><td><strong>pythnet</strong></td><td><a href="blockchain-environments/solana/#pythnet">#pythnet</a></td><td><a href=".gitbook/assets/chain-icons/pythnet.svg">pythnet.svg</a></td></tr><tr><td><strong>Rootstock</strong></td><td><a href="blockchain-environments/evm/#rootstock">#rootstock</a></td><td><a href=".gitbook/assets/chain-icons/rootstock.svg">rootstock.svg</a></td></tr><tr><td><strong>Scroll</strong></td><td><a href="blockchain-environments/evm/#scroll">#scroll</a></td><td></td></tr><tr><td><strong>Seda</strong></td><td><a href="blockchain-environments/cosmwasm/#seda">#seda</a></td><td></td></tr><tr><td><strong>Sei</strong></td><td><a href="blockchain-environments/cosmwasm/#sei">#sei</a></td><td><a href=".gitbook/assets/chain-icons/sei.svg">sei.svg</a></td></tr><tr><td><strong>Ethereum Sepolia</strong></td><td><a href="blockchain-environments/evm/#sepolia">#sepolia</a></td><td><a href=".gitbook/assets/chain-icons/sepolia.svg">sepolia.svg</a></td></tr><tr><td><strong>Solana</strong></td><td><a href="blockchain-environments/solana/#solana">#solana</a></td><td><a href=".gitbook/assets/chain-icons/solana.svg">solana.svg</a></td></tr><tr><td><strong>Stargaze</strong></td><td><a href="blockchain-environments/cosmwasm/#stargaze">#stargaze</a></td><td><a href=".gitbook/assets/chain-icons/stargaze.svg">stargaze.svg</a></td></tr><tr><td><strong>Sui</strong></td><td><a href="blockchain-environments/sui/#sui">#sui</a></td><td><a href=".gitbook/assets/chain-icons/sui.svg">sui.svg</a></td></tr><tr><td><strong>Terra</strong></td><td><a href="blockchain-environments/cosmwasm/#terra">#terra</a></td><td><a href=".gitbook/assets/chain-icons/terra.svg">terra.svg</a></td></tr><tr><td><strong>Terra2</strong></td><td><a href="blockchain-environments/cosmwasm/#terra2">#terra2</a></td><td><a href=".gitbook/assets/chain-icons/terra2.svg">terra2.svg</a></td></tr><tr><td><strong>Xpla</strong></td><td><a href="blockchain-environments/cosmwasm/#xpla">#xpla</a></td><td><a href=".gitbook/assets/chain-icons/xpla.svg">xpla.svg</a></td></tr></tbody></table>
