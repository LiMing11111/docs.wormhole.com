# 专用中继器

![专业中继器](../../.gitbook/assets/specialized-relayer.png)

Wormhole 与许多[生态系统](../../blockchain-environments/)兼容，集成非常简单。

## 链上

为了在不同链之间发送和接收信息，了解一些[链上组件](../../tan-suo-chong-dong-wormhole/components.md#on-chain-components)是很重要的。&#x20;

### 发送信息

要发送信息，无论环境或链如何，核心合约都要调用来自 [emitter](../../reference/glossary.md#emitter) 的信息参数。

该 `emitter` 可能事你的合约或现有的应用程序，如 [Token Bridge](https://github.com/wormhole-foundation/wormhole/blob/main/whitepapers/0003\_token\_bridge.md)，或 [NFT Bridge](https://github.com/wormhole-foundation/wormhole/blob/main/whitepapers/0006\_nft\_bridge.md)。

{% tabs %}
{% tab title="EVM" %}
使用 `IWormhole` 接口（[source](https://github.com/wormhole-foundation/wormhole/blob/main/ethereum/contracts/interfaces/IWormhole.sol)），我们可以直接向[核心合约](../../tan-suo-chong-dong-wormhole/core-contracts.md)发布信息。

```solidity
// ...

IWormhole wormhole = IWormhole(wormholeAddr);

// Get the fee for publishing a message
uint256 wormholeFee = wormhole.messageFee();

// ... fee/value checks

// create the HelloWorldMessage struct
HelloWorldMessage memory parsedMessage = HelloWorldMessage({
    payloadID: uint8(1),
    message: helloWorldMessage
});

// encode the HelloWorldMessage struct into bytes
bytes memory encodedMessage = encodeMessage(parsedMessage);

// Send the HelloWorld message by calling publishMessage on the
// Wormhole core contract and paying the Wormhole protocol fee.
messageSequence = wormhole.publishMessage{value: wormholeFee}(
    0, // batchID
    encodedMessage,
    wormholeFinality()
);
```

更多详情请参阅[示例源码](https://github.com/wormhole-foundation/wormhole-scaffolding/blob/main/evm/src/01\_hello\_world/HelloWorld.sol)
{% endtab %}

{% tab title="Solana" %}
使用 `wormhole_anchor_sdk::wormhole` 模块并给定 wormhole 程序账户，我们可以直接向核心合约传递信息。

```rust
// ...

let fee = ctx.accounts.wormhole_bridge.fee();
// ... fee check/send

let config = &ctx.accounts.config;
let payload: Vec<u8> = HelloWorldMessage::Hello { message }.try_to_vec()?;

// Invoke `wormhole::post_message`.
wormhole::post_message(
    CpiContext::new_with_signer(
        ctx.accounts.wormhole_program.to_account_info(),
        wormhole::PostMessage {
            // ... set fields
        },
        &[
            // ... set seeds
        ],
    ),
    config.batch_id,
    payload,
    config.finality.into(),
)?;

// ...
```

更多详情请参阅[示例源码](https://github.com/wormhole-foundation/wormhole-scaffolding/blob/main/solana/programs/01\_hello\_world/src/lib.rs)
{% endtab %}
{% endtabs %}

一旦信息从核心合约中发出，[Guardian Network](../../tan-suo-chong-dong-wormhole/guardian.md) 就会发现信息并签署认证摘要（[VAA](../../tan-suo-chong-dong-wormhole/vaa.md)）。我们将在下面的[链下](specialized-relayer.md#off-chain)部分更深入的讨论这一点。

{% hint style="info" %}
默认情况下， VAAs 是 [multicast](../../tan-suo-chong-dong-wormhole/core-contracts.md#multicast)。这意味着给定信息没有默认**目标链**。信息的格式和接收时的处理方式，由应用程序开发人员决定。&#x20;
{% endhint %}

### 接收信息

接收信息的方式取决于环境。

{% tabs %}
{% tab title="EVM" %}
在 EVM 链上，传递的信息是二进制编码的原始 VAA。

核心合约尚未对其进行验证，在调用 `parseAndVerifyVM` 之前，应将其视为不可信任的输入。

```solidity
function receiveMessage(bytes memory encodedMessage) public {
    // call the Wormhole core contract to parse and verify the encodedMessage
    (
        IWormhole.VM memory wormholeMessage,
        bool valid,
        string memory reason
    ) = wormhole().parseAndVerifyVM(encodedMessage);

    // ... safety checks

    // decode the message payload into the HelloWorldMessage struct
    HelloWorldMessage memory parsedMessage = decodeMessage(
        wormholeMessage.payload
    );

    // ... do something cool
}
```

更多详情请参阅[示例源码](https://github.com/wormhole-foundation/wormhole-scaffolding/blob/main/evm/src/01\_hello\_world/HelloWorld.sol)
{% endtab %}

{% tab title="Solana" %}
On Solana, the VAA is first posted and verified by the core contract, after which it can be read by the receiving contract and action taken.

```rust
pub fn receive_message(ctx: Context<ReceiveMessage>, vaa_hash: [u8; 32]) -> Result<()> {
    let posted_message = &ctx.accounts.posted;

    if let HelloWorldMessage::Hello { message } = posted_message.data() {
        // ... check message
        // ... do something cool 
        // Done
        Ok(())
    } else {
        Err(HelloWorldError::InvalidMessage.into())
    }
}

```

More details in [Example Source](https://github.com/wormhole-foundation/wormhole-scaffolding/blob/main/solana/programs/01\_hello\_world/src/lib.rs)
{% endtab %}
{% endtabs %}

除了应执行的特定环境检查外，合同还应注意检查[正文](../../tan-suo-chong-dong-wormhole/vaa.md#body)中的其他字段，例如：

* **Emitter：**这是否来自我期望的 emitter 地址和链 id？通常，合约将提供一种方法来注册新的 emitter 并根据其信任的 emitter 集检查传入的消息。
* 序列：这是我期望的序列号吗？我应该如何处理无序交付？
* **Consistency Level**：对于这条信息来自的链，[consistency level](../../tan-suo-chong-dong-wormhole/core-contracts.md#consistencylevel) 是否足以保证我采取某些操作后交付不会被撤销？

VAA 主体之外但也与之相关的是 VAA 的摘要，通过检查摘要是否已经被看到，可以将其用于重放保护。

由于 payload 本身是特定于应用程序的，因此可能还有其他因素需要检查以确保安全。

## 链下

为了在链之间传递信息，需要一些[链下进程](../../tan-suo-chong-dong-wormhole/components.md#off-chain-components)。[Guardians](../../tan-suo-chong-dong-wormhole/guardian.md) 观察核心合约中发生的事件并签署 [VAA](../../tan-suo-chong-dong-wormhole/vaa.md)。

在足够多的 Guardians 签署信息后（至少 `2/3 + 1` 或 13 / 19 个 guardians），VAA 就可以被传递到目标链。is available to be

一旦 VAA 可用，[中继器](../../tan-suo-chong-dong-wormhole/relayer.md)就可以通过格式正确的交易将其传送到目标链。

需要中继器将包含信息的 VAA 传递到目标链。当中继器专门为自定义应用程序编写时，则被称为专用中继器[专用中继器](specialized-relayer.md)

专门的中继器可以是一个简单的浏览器进程，它在提交交易后轮询 [API](../../reference/api-docs/) 以获取 VAA 的可用性，并将其传递到目标链。也可以使用 [Spy](../../tan-suo-chong-dong-wormhole/spy.md) 和一些守护进程来实现，这些守护进程会监听相关 `chainID` 和 `emitter` 的VAAs，一旦检测到这样的信息，就会采取相应的行动。

#### 简易中继器

无论环境如何，为了获得我们想要中继的 VAA，我们需要：

1. `emitter` 地址
2. 我们感兴趣的消息的 `sequence` id
3. 发出信息的链的 `chainId`

有了这三个组件，我们就能唯一识别 VAA 并从 [API](../../reference/api-docs/) 中获取它。

**获取 VAA**

使用 [SDK](../../reference/sdk-docs/) 中提供的 `getSignedVAAWithRetry` 函数，我们可以轮询 Guardian RPC 直到签名的 VAA 准备就绪。

```ts
import { 
    getSignedVAAWithRetry, 
    parseVAA, 
    CHAIN_ID_SOLANA,
    CHAIN_ID_ETH,
} from "@certusone/wormhole-sdk";


const RPC_HOSTS = [ /* ...*/ ]

async function getVAA(emitter: string, sequence: string, chainId: number): Promise<Uint8Array> {
    // Wait for the VAA to be ready and fetch it from 
    // the guardian network
    const {vaaBytes} = await getSignedVAAWithRetry(
        RPC_HOSTS,
        chainId,
        emitter,
        sequence
    )
    return vaaBytes
}

const vaaBytes = await getVAA('0xdeadbeef', 1, CHAIN_ID_ETH);
```

一旦我们获得了 VAA，交付方式就取决于链。

{% tabs %}
{% tab title="EVM" %}
在 EVM 链上，VAA 的字节可以直接作为参数传递给 ABI 方法。

```ts
// setup eth wallet
const ethProvider = new ethers.providers.StaticJsonRpcProvider(ETH_HOST);
const ethWallet = new ethers.Wallet(WALLET_PRIVATE_KEY, ethProvider);

// create client to interact with our target app
const ethHelloWorld = HelloWorld__factory.connect(
    '0xbeefdead',
    ethWallet
);

// invoke the receiveMessage on the ETH contract
// and wait for confirmation
const receipt = await ethHelloWorld
    .receiveMessage(vaaBytes)
    .then((tx: ethers.ContractTransaction) => tx.wait())
    .catch((msg: any) => {
        console.error(msg);
        return null;
    });
```
{% endtab %}

{% tab title="Solana" %}
```ts

import {CONTRACTS} from '@certusone/wormhole-sdk'

export const WORMHOLE_CONTRACTS = CONTRACTS[NETWORK];
export const CORE_BRIDGE_PID = new PublicKey(WORMHOLE_CONTRACTS.solana.core);

// First, post the VAA to the core bridge
await postVaaSolana(
    connection,
    wallet.signTransaction,
    CORE_BRIDGE_PID,
    wallet.key(),
    vaaBytes 
);

const program = createHelloWorldProgramInterface(connection, programId);
const parsed = isBytes(wormholeMessage)
    ? parseVaa(wormholeMessage)
    : wormholeMessage;

const ix = program.methods
    .receiveMessage([...parsed.hash])
    .accounts({
        payer: new PublicKey(payer),
        config: deriveConfigKey(programId),
        wormholeProgram: new PublicKey(wormholeProgramId),
        posted: derivePostedVaaKey(wormholeProgramId, parsed.hash),
        foreignEmitter: deriveForeignEmitterKey(programId, parsed.emitterChain),
        received: deriveReceivedKey(
        programId,
        parsed.emitterChain,
        parsed.sequence
        ),
    })
    .instruction();

const transaction = new Transaction().add(ix);
const { blockhash } = await connection.getLatestBlockhash(commitment);
transaction.recentBlockhash = blockhash;
transaction.feePayer = new PublicKey(payerAddress);

const signed = await wallet.signTxn(transaction);
const txid = await connection.sendRawTransaction(signed);

await connection.confirmTransaction(txid);

```
{% endtab %}
{% endtabs %}

参阅[专用中继器教程](../../tutorials/app-integration/specialized-relayer.md)以获取详细的演示。

## 参考

点击[这里](../../tan-suo-chong-dong-wormhole/components.md)了解更多有关架构和核心组件的信息
