# 引导

## Wormhole Gateway 使用指南

{% hint style="info" %}
以下内容是给任何希望启用来自Gateway的桥接功能的Cosmos链开发者看的，Gateway是一个利用Wormhole Guardian网络实现从以太坊到Cosmos轻松桥接的Wormhole Cosmos链。: [The Gateway to Cosmos](https://wormhole.com/gateway/)
{% endhint %}

## 步骤1. 建议将您的链添加到 Wormhole Guardians

1. 在Wormhole Gateway下通过填写Cosmos链治理提案模板开启一个新的GitHub讨论 - [here](https://github.com/wormhole-foundation/wormhole/discussions/new?category=gateway)
2. 允许96小时进行讨论和治理投票。

## 步骤 2. 加入 Wormhole Discord

1. 加入 wormhole discord ([link](https://discord.gg/wormholecrypto))
2. 向版主Susu (`susu.wormhole`)发送消息, 以便加入 `#guardian-cosmos` 频道。

## 步骤3. 建立一个IBC连接

1.  把你的 IBC relayer(s) 添加进 Wormhole Gateway 的白名单。

    1. IBC中继器应通过 `wormchaind` CLI 生成一个地址 - [here](https://github.com/wormhole-foundation/wormhole/tree/main/wormchain).
    2. 填写 [IBC relayer allowlist request template](onboard.md#ibc-relayer-allowlist-request-template).
    3. 在 `#guardian-cosmos` 频道发布请求。


2. Establish the IBC connection.
   1. Please ensure that the `trusting_period` and `trust_threshold` parameters are set to the safest values. E.g. `trust_threshold` should be 2/3 and `trusting_period` should be 2/3 the unbonding period of your chain.
   2. See an example IBC relayer config for Wormhole Gateway [below](onboard.md#wormhole-gateway-ibc-relayer-config).
   3. Please see docs [here](https://github.com/wormhole-foundation/wormhole/blob/main/wormchain/syncing.md) on how to set up your own Wormhole Gateway node to connect your IBC relayer to. Or, you can see available public nodes on the [cosmos chain registry](https://github.com/cosmos/chain-registry/blob/master/gateway/chain.json).
3. Share the IBC connection details in the `#guardian-cosmos` channel along with a request to the Wormhole Contributors to prepare governance for the IBC connection.
   1. Allow 48 hours for governance vote on accepting this IBC channel.

## Step 4. \[Optional] UI Integration with Wormhole Connect

[Wormhole Connect](https://wormhole.com/connect/) is a seamless way to embed bridging directly to your app with 3 lines of code. [Integrating Connect](https://wormhole-connect-builder.netlify.app/) is fast, customizable, and brings all the functionality and utility of Wormhole right into your own application.

Please refer to these reference PRs to add your Cosmos chain into Wormhole Connect. Your PRs will need to be reviewed and merged by Wormhole Core Contributors.

1. Add your Cosmos chain ID to the Wormhole SDK: [\[sdk/js\] Add Kujira chain id by M-Picco · Pull Request #3381 · wormhole-foundation/wormhole (github.com)](https://github.com/wormhole-foundation/wormhole/pull/3381/files)
2. Add your Cosmos chain to Wormhole Connect: [Add kujira chain by M-Picco · Pull Request #1009 · wormhole-foundation/wormhole-connect (github.com)](https://github.com/wormhole-foundation/wormhole-connect/pull/1009/files)

## Step 5. Add bridged assets to the Cosmos Chain Registry and other relevant wallet and frontend registries

1. Permissionlessly attest the assets you would like to bridge into your chain (if not already attested) to Wormhole Gateway.
2. Raise relevant PRs to ensure that explorers, wallets, and other UIs recognize the Wormhole assets when they are bridged to your chain.
   1. Example PR adding Wormhole assets to Osmosis Mintscan ([example](https://github.com/cosmostation/chainlist/pull/865)).

{% hint style="success" %}
🎉 Congratulations! You’ve successfully connected your Cosmos chain to Gateway. If you have any questions or concerns, please reach out to Susu on the Wormhole Discord.
{% endhint %}

## IBC Relayer Allowlist Request Template

```
Hey @Guardians! Thank you for passing governance to support **[Cosmos Chain]** via Wormhole Gateway. We are very excited to integrate with Wormhole!

We will be using **[Relayer Provider]** as our IBC relayer to support the connection to Wormhole Gateway. Their address is **[Wormhole Gateway address].** 

Could one of the Guardians please allowlist this address so that it can submit transactions to Wormhole Gateway?

We understand that if this address misbehaves, the sponsoring Guardian can remove it from the allowlist at any time, which would effectively shut down IBC bridging to/from our chain and Gateway.

Thank you!
```

## Wormhole Gateway IBC Relayer Config

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
