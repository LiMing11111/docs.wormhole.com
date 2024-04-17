# Gateway

### 概览

Wormhole Gateway 是一个基于 Cosmos-SDK 的链，提供了一种将非原生资产桥接到 Cosmos 生态系统中的方式，并作为 Cosmos 链间统一流动性的来源。

{% hint style="success" %}
因为使用 IBC（Inter-Blockchain Communication）从 Gateway 桥接资产到 Cosmos 链，避免了流动性的碎片化，并且通过 Wormhole 桥接到 Cosmos 生态系统中的外部资产在 Cosmos 的各个链之间能够实现流动性的统一。
{% endhint %}

除了促进资产转移外，Wormhole Gateway（前称 `wormchain`，又名 `Shai-Hulud`）还允许 Wormhole 与 [accountant](https://github.com/wormhole-foundation/wormhole/blob/main/whitepapers/0011\_accountant.md) 确保正确的账务处理。

### 详细说明

Wormhole Gateway 是作为一组合约和模块实现的。&#x20;

这些组件的合约地址是：

| 合约                    | 主网地址                                                                  | 测试网地址                                                                 |
| --------------------- | --------------------------------------------------------------------- | --------------------------------------------------------------------- |
| Wormhole core bridge  | `wormhole1ufs3tlq4umljk0qfe8k5ya0x6hpavn897u2cnf9k0en9jr7qarqqaqfk2j` | `wormhole16jzpxp0e8550c9aht6q9svcux30vtyyyyxv5w2l2djjra46580wsazcjwp` |
| Wormhole token bridge | `wormhole1466nf3zuxpya8q9emxukd7vftaf6h4psr0a07srl5zw74zh84yjq4lyjmh` | `wormhole1aaf9r6s7nxhysuegqrxv0wpm27ypyv4886medd3mrkrw6t4yfcnst3qpex` |
| IBC Translator        | `wormhole14ejqjyq8um4p3xfqj74yld5waqljf88fz25yxnma0cngspxe3les00fpjx` | `wormhole1ctnjk7an90lz5wjfvr3cf6x984a8cjnv8dpmztmlpcq4xteaa2xs9pwmzk` |

#### Wormhole 核心合约

在需要通用消息传递的每个 Cosmos 链上，仍需要部署用于发送消息和验证 [Guardian](../guardian.md) 签名的[核心合约](https://app.gitbook.com/o/SK9htrrKEoxzXNcFitC5/s/uLLPBOibnlYmSJX40vn0/)。值得注意的是，对于 Gateway 代币桥接，不需要部署任何核心合约。

#### IBC Shim 合约

一个 CosmWasm 合约，通过在 Wormhole 和 IBC 消息格式之间进行转换，处理进入和离开 Cosmos 生态系统的桥接。它维护了一个从 `chainId -> channelId` 的映射，用于白名单上的 IBC 通道以发送和接收数据包。

该合约通过接收 [Contract Controlled Transfer VAAs](../vaa.md#token--message) 支持转入 Cosmos 生态系统的转账。

这类转账的逻辑流程如下：&#x20;

* 在  [Token Bridge](../core-contracts.md#token-bridge) 上兑换 VAA&#x20;
* 铸造 [Token Factory](./#token-factory-module) 代币&#x20;
* 解码附加的负载作为 [`GatewayIbcTokenBridgePayload`](./#gatewayibctokenbridgepayload)
* 通过 IBC 将代币发送到目的地 Cosmos 链&#x20;

该合约还通过实现一个 `execute` 处理器来支持从 Cosmos 生态系统转出，该处理器接受一个 [`GatewayIbcTokenBridgePayload`](./#gatewayibctokenbridgepayload) 和一定数量的 Token Factory 代币（作为要桥接出去的代币）在 `info.funds` 中。&#x20;

这类转账的逻辑流程如下：

* 销毁 [Token Factory](./#token-factory-module) 代币&#x20;
* 解锁 CW20 代币&#x20;
* 授权 [Token Bridge](../core-contracts.md#token-bridge) 使用 CW20 代币&#x20;
* 根据 [`GatewayIbcTokenBridgePayload`](./#gatewayibctokenbridgepayload) 是 `Simple` 类型还是 `ContractControlled` 类型，调用 `InitiateTransfer` 或 `InitiateTransferWithPayload`

#### Token Factory 模块

在 Wormhole Gateway 上部署标准的 [Token Factory](https://github.com/CosmosContracts/juno/tree/v14.1.1/x/tokenfactory) 模块，以创建新的代币。&#x20;

#### IBC Composability 中间件

IBC Composability 中间件位于 [PFM (Packet Forwarding Module)](https://github.com/strangelove-ventures/packet-forward-middleware) 和 IBC Hooks 中间件之上，将这两者结合起来。它使 Cosmos 链上的集成者能够使用单一的负载结构支持 `Cosmos -> Cosmos` 和 `Cosmos -> External`的流程。

它接受一个 [`GatewayIbcTokenBridgePayload`](./#gatewayibctokenbridgepayload) 负载，并通过查看负载中的 `chainId` 来决定是调用 PFM 还是 IBC Hooks 中间件。&#x20;

1. 如果 `chainId` 是一个支持 IBC 的链，它将为 PFM 格式化一个负载，以将 ICS20 转账转发到目的地支持 IBC 的链。&#x20;
2. 如果 `chainId` 是一个外部链，它将为 IBC Hooks 中间件格式化一个负载，以调用 IBC Shim 合约的 `execute` 处理器以实现桥接出去。

#### IBC Hooks 中间件

在 Wormhole Gateway 上部署 [IBC Hooks 中间件](https://github.com/osmosis-labs/osmosis/tree/v15.2.0/x/ibc-hooks)，允许 ICS-20 代币转移同时启动合约调用。&#x20;

### 集成

通过几行代码即可实现与 Wormhole Gateway 的集成，支持以下功能：&#x20;

* 从外部链到任何支持的 Cosmos 链的转移，见 [Into Cosmos](./#into-cosmos)
* 从任何支持的 Cosmos 链到外部链的转移，见 [Out of Comsos](./#out-of-cosmos)
* 在任何支持的 Cosmos 链之间的转移，见 [Between Cosmos Chains](./#between-cosmos-chains)

#### Into Cosmos

为了将资产桥接进入一个 Cosmos 链，将在外部链上启动一个资产转移，其 [payload](./#gatewayibctokenbridgepayload) 由 Gateway 或更具体地说是 [IBC Shim Contract](./#ibc-shim-contract) 理解。&#x20;

一旦在 Gateway 收到，该资产的 CW20 表现形式将通过 IBC 使用已建立的 [ICS20 protocol](https://github.com/cosmos/ibc/tree/main/spec/app/ics-020-fungible-token-transfer) 发送到目的地链。&#x20;

使用 [SDK](../../reference/sdk-docs/) 的一个示例：

```ts
import * as wh from '@certusone/wormhole-sdk';

// ...

const transferDetails = {
  gateway_transfer: {                               // This is a simple transfer, no additional payload 
    chain: 4000,                                    // Chain Id of the Cosmos chain we're sending to
    recipient: "<cosmos-chain-recipient-address>",  // Address of recipient (base64 encoded bech32)
    fee: 0,                                         // Fee for transfer (0 for now)
    nonce: 0,                                        
  }
}

const ibcTranslatorAddress = "wormhole14ejqjyq8um4p3xfqj74yld5waqljf88fz25yxnma0cngspxe3les00fpjx"
// Convert the transfer details to a Uint8Array for sending
const payload = new Uint8Array(Buffer.from(JSON.stringify(transferDetails)))

// Send transfer transaction on Ethereum
await txReceipt = wh.transferFromEth(
  wh.consts.TESTNET.eth.token_bridge // source chain token bridge address
  wallet,                            // signer for eth tx
  "0xdeadbeef...",                   // address of token being transferred
  10000000n,                         // amount of token in its base units
  wh.consts.CHAINS.wormchain,        // chain id we're sending to
  ibcTranslatorAddress,              // The address of the ibc-translator contract on the Gateway
  0,                                 // relayer fee, 0 for now
  {},                                // tx overrides (gas fees, etc...)
  payload                            // The payload Gateway uses to route transfers
);

// ...
```

#### Out of Cosmos

为了将资产从 Cosmos 生态系统桥接出去或在 Cosmos 链之间进行桥接，将在源链上启动一个 IBC 转账，转账到 Gateway，并在 `memo` 字段中包含有关转账的详细信息。

例如，使用 [cosmjs](https://github.com/cosmos/cosmjs)：

```ts
const wallet = await DirectSecp256k1HdWallet.fromMnemonic(faucet.mnemonic);
const client = await SigningStargateClient.connectWithSigner(
  simapp.tendermintUrl,
  wallet,
  defaultSigningClientOptions
);

const memo = JSON.stringify({
    gateway_ibc_token_bridge_payload:{
        gateway_transfer:{
            chain:     0,   // chain id of receiver
            recipient: "",  // address of receiver
            fee:       0,   // fee to cover transfer
            nonce:     0,   // 
        }
    }
})
const ibcTranslatorAddress = "wormhole14ejqjyq8um4p3xfqj74yld5waqljf88fz25yxnma0cngspxe3les00fpjx"
const result = await client.sendIbcTokens(
  faucet.address0,     // sender address
  ibcTranslatorAddress,// receiver address
  coin(1234, "ucosm"), // amount and coin
  "transfer",          // source port
  "channel-2186",      // source channel, TODO: fill in once deployed
  timeoutHeight,       // 
  timeoutTimestamp,    // 
  0,                   // fee to cover transaction 
  memo                 // formatted payload with details about transfer
);
```

#### Between Cosmos Chains

从实现的角度看，Cosmos 链之间的转移与从 [Cosmos 桥接出去](./#out-of-cosmos)的操作完全相同。不同之处在于传递的 `chain` ID 是一个 Cosmos 链。

### 数据结构

由 Gateway 协议使用的核心数据结构。&#x20;

#### GatewayIbcTokenBridgePayload

代币转移的核心数据结构是 `GatewayIbcTokenBridgePayload`，其中包含了 Gateway 用来执行转移的转移详情。

```rust
pub enum GatewayIbcTokenBridgePayload {
    GatewayTransfer {
        chain: u16,
        recipient: Binary,
        fee: u128,
        nonce: u32,
    },
    GatewayTransferWithPayload {
        chain: u16,
        contract: Binary,
        payload: Binary,
        nonce: u32,
    },
}
```

当发送 `GatewayIbcTokenBridge` 负载时，必须将其序列化为 json 格式。 为了正确的 json 编码，`Binary` 值需采用 base64 编码。&#x20;

Cosmos 链的 `recipient` 地址也是采用 base64 编码的 bech32 地址。例如，如果 `recipient` 是 `wormhole1f3jshdsmzl03v03w2hswqcfmwqf2j5csw223ls`，则编码将是 `d29ybWhvbGUxZjNqc2hkc216bDAzdjAzdzJoc3dxY2Ztd3FmMmo1Y3N3MjIzbHM=` 的直接 base64 编码。&#x20;

`chain` 的值对应于 [Wormhole chain IDs](../../reference/glossary.md#chain-id)。

`fee` 和 `nonce` 都是 Wormhole 特有的参数，但两者现在都未使用。

对于从 Cosmos/IBC 链来的传入 IBC 消息，`receiver` 字段将在 `Simple.recipient` 字段中使用 base64 编码，并且 `channel-id` 将作为等效的 wormhole `chain` id 包含进来。

### 费用结构

使用 Gateway 的费用很低。目前，源链的 gas 费是唯一的成本。

#### 需要支付的费用

* 源链气体费用：必须支付源链（例如以太坊）的 Gas 费用。&#x20;
* 中继费用 \[源链 => Gateway]：处理 Wormhole 消息的成本。当前为 0，但将来可能会变化。
* 目的链 Gas 费用 \[非 Cosmos]：目的链（例如以太坊）的 Gas 费用必须由中继方或在手动赎回的情况下由用户支付。

#### 不需要支付的费用

* Gateway：Gateway 没有基于代币定价的计量或要求用户支付 gas 费用。&#x20;
* 中继费用 \[Gateway => Cosmos]：中继者不通过用户费用获得激励。&#x20;
* 目的链 \[Cosmos]：IBC 中继者承担目的链上的处理成本。

### 另见

[Gateway 区块浏览器](https://bigdipper.live/wormhole)&#x20;

当然，Wormhole Gateway 是开源的，源代码可在[此处](https://github.com/wormhole-foundation/wormhole/tree/main/wormchain)获取。&#x20;

实现这些功能的合约在[此处](https://github.com/wormhole-foundation/wormhole/tree/gateway-integration/cosmwasm/contracts)。
