# 引导

## Wormhole Gateway 使用指南

{% hint style="info" %}
以下内容是给任何希望启用来自 Gateway 的桥接功能的 Cosmos 链开发者看的，Gateway 是一个利用 Wormhole Guardian 网络实现从以太坊到 Cosmos 轻松桥接的Wormhole Cosmos 链 : [The Gateway to Cosmos](https://wormhole.com/gateway/)
{% endhint %}

## 步骤 1. 建议将您的链添加到 Wormhole Guardians

1. 在 Wormhole Gateway 下通过填写 Cosmos 链治理提案模板开启一个新的 GitHub 讨论 - [这里](https://github.com/wormhole-foundation/wormhole/discussions/new?category=gateway)
2. 允许 96 小时进行讨论和治理投票。

## 步骤 2. 加入 Wormhole Discord

1. 加入 wormhole discord ([link](https://discord.gg/wormholecrypto))
2. 向版主 Susu (`susu.wormhole`) 发送消息, 以便加入 `#guardian-cosmos` 频道。

## 步骤 3. 建立一个 IBC 连接

1. 将您的 IBC relayer(s) 添加到 Wormhole Gateway 的白名单。
   1. IBC 中继器应通过 `wormchaind` CLI 生成一个地址 - [这里](https://github.com/wormhole-foundation/wormhole/tree/main/wormchain).
   2. 填写 [IBC relayer allowlist request template](onboard.md#ibc-relayer-allowlist-request-template).
   3. 在`#guardian-cosmos` 频道发布请求。
2. 建立 IBC 连接。
   1. 请确保设置`trusting_period`和`trust_threshold`参数到最安全的值。例如，`trust_threshold` 应为 2/3，`trusting_period` 应为您链的解绑期的 2/3。
   2. 参见 Wormhole Gateway 的 IBC 中继器配置示例 [下方](onboard.md#wormhole-gateway-ibc-zhong-ji-qi-pei-zhi)。
   3. 请查看[此处](https://github.com/wormhole-foundation/wormhole/blob/main/wormchain/syncing.md)的文档了解如何设置您自己的 Wormhole Gateway 节点以连接您的 IBC 中继器。或者，您可以在 [cosmos 链注册](https://github.com/cosmos/chain-registry/blob/master/gateway/chain.json)上查看可用的公共节点。
3. 在`#guardian-cosmos`频道共享 IBC 连接细节，并请求 Wormhole 贡献者准备 IBC 连接的治理。
   1. 允许 48 小时进行接受此 IBC 通道的治理投票。

## 步骤 4. \[可选] 使用 Wormhole Connect 集成 UI

[Wormhole Connect](https://wormhole.com/connect/) 是一种将桥接功能直接嵌入到您的应用中的方式，仅需 3 行代码。[集成 Connect](https://wormhole-connect-builder.netlify.app/) 快速、可定制，并将 Wormhole 的所有功能和实用性直接带入您自己的应用程序。

请参考这些参考 PR 将您的 Cosmos 链添加到 Wormhole Connect。您的 PR 需要由 Wormhole 核心贡献者审查和合并。

1. 将您的 Cosmos 链 ID 添加到 Wormhole SDK：[PR #3381 · wormhole-foundation/wormhole (github.com)](https://github.com/wormhole-foundation/wormhole/pull/3381/files)
2. 将您的 Cosmos 链添加到 Wormhole Connect：[PR #1009 · wormhole-foundation/wormhole-connect (github.com)](https://github.com/wormhole-foundation/wormhole-connect/pull/1009/files)

## 步骤 5. 将桥接资产添加到 Cosmos 链注册表及其他相关钱包和前端注册表

1. 无需许可地证明您希望桥接到您链上的资产（如果尚未证明）至 Wormhole Gateway。
2. 提交相关 PR 以确保浏览器、钱包和其他 UI 在您的链上桥接 Wormhole 资产时能够识别这些资产。
   1. 示例 PR，将 Wormhole 资产添加到 Osmosis Mintscan（[示例](https://github.com/cosmostation/chainlist/pull/865)）。

{% hint style="success" %}
🎉 恭喜！您已成功将您的 Cosmos 链连接到 Gateway。如果您有任何问题，请联系Wormhole Discord 上的 Susu。
{% endhint %}

## IBC Relayer Allowlist Request Template

```
Hey @Guardians! Thank you for passing governance to support **[Cosmos Chain]** via Wormhole Gateway. We are very excited to integrate with Wormhole!

We will be using **[Relayer Provider]** as our IBC relayer to support the connection to Wormhole Gateway. Their address is **[Wormhole Gateway address].** 

Could one of the Guardians please allowlist this address so that it can submit transactions to Wormhole Gateway?

We understand that if this address misbehaves, the sponsoring Guardian can remove it from the allowlist at any time, which would effectively shut down IBC bridging to/from our chain and Gateway.

Thank you!
```

## Wormhole Gateway IBC 中继器配置

```toml
[global]
log_level = "info"

[mode.clients]
enabled = true
refresh = true
misbehaviour = false

[mode.connections]
enabled = false

[mode.channels]
enabled = false

[mode.packets]
enabled = true
clear_interval = 50
clear_on_start = true
tx_confirmation = true
auto_register_counterparty_payee = false

[rest]
enabled = true
host = "127.0.0.1"
port = 3000

[telemetry]
enabled = true
host = "127.0.0.1"
port = 3001

[telemetry.buckets.latency_submitted]
start = 500
end = 20000
buckets = 10

[telemetry.buckets.latency_confirmed]
start = 1000
end = 30000
buckets = 10

[[chains]]
id = "wormchain"
type = "CosmosSdk"
rpc_addr = "..."
grpc_addr = "..."
rpc_timeout = "10s"
trusted_node = true
account_prefix = "wormhole"
key_name = "default"
key_store_type = "Test"
store_prefix = "ibc"
default_gas = 1000000
max_gas = 9000000
gas_multiplier = 1.2
max_msg_num = 30
max_tx_size = 180000
max_grpc_decoding_size = 33554432
clock_drift = "5s"
max_block_time = "30s"
ccv_consumer_chain = false
memo_prefix = ""
sequential_batch_tx = false
trusting_period = '14days'

[chains.event_source]
mode = "push"
url = "..."
batch_delay = "500ms"

[chains.trust_threshold]
numerator = "2"
denominator = "3"

[chains.gas_price]
price = 0.0
denom = "utest"

[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-3'], # osmosis transfer
]

[chains.address_type]
derivation = "cosmos
```
