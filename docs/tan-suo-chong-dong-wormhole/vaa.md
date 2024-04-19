# VAAs（已验证的操作批准）

VAAs 是 Wormhole 中的核心消息原语。您可以将其视为跨链数据包，这些数据包在跨链应用合约与核心合约互动时发出。

基本的 VAA 包括两个部分 —— 头部（Header）和主体（Body）。

由合约发出的消息需要由守护者（guardians）验证后，才能发送到目标链。一旦大多数守护者观察到消息并且达到了最终性（finality），守护者们会[对消息主体的 keccak256 哈希进行签名](vaa.md#signatures)。

消息被封装在一个称为 VAA 的结构中，该结构将消息与守护者签名结合起来，形成一个证明。

VAAs 通过 (`emitter_chain`, `emitter_address`, `sequence`) 元组进行唯一索引。通过这些信息查询守护者 [RPC](../reference/sdk-docs/#mainnet-guardian-rpc) 或 [API](../reference/api-docs/) 可以获得 VAA。

这些 VAA 最终是接收链上智能合约必须处理的，以接收 Wormhole 消息。

### VAA 格式

VAA 包含两个数据部分。

#### Header（头部）

头部包含当前 VAA 的元数据，当前活跃的守护者集合，以及迄今为止收集的签名列表。

```rust
byte        version                  // VAA Version
u32         guardian_set_index       // Indicates which guardian set is signing
u8          len_signatures           // Number of signatures stored
[]signature signatures               // Collection of guardian signatures
```

其中每个`signature`是：

```rust
u8          index       // The index of this guardian in the guardian set
[65]byte    signature   // The ECDSA signature 
```

#### Body（主体）

主体是从链上消息确定性地派生的。任何两个守护者必须派生出完全相同的主体。这一要求的存在是为了确保每个 VAA 和消息之间始终是一对一的关系，以避免消息的双重处理。

```rust
u32         timestamp       // The timestamp of the block this message was published in
u32         nonce           //  
u16         emitter_chain   // The id of the chain that emitted the message
[32]byte    emitter_address // The contract address (wormhole formatted) that called the core contract
u64         sequence        // The auto incrementing integer that represents the number of messages published by this emitter
u8          consistency_level // The consistency level (finality) required by this emitter
[]byte      payload         // arbitrary bytes containing the data to be acted on
```

主体是消费者相关的信息，可以通过如 parseAndVerifyVAA 的调用返回。因为 emitterAddress 包含在主体中，开发者可以判断此 VAA 是否源自一个可信合约。

{% hint style="warning" %}
VAAs 是设计成无特定目的地的，因此它们在 Wormhole 网络中是以多播的方式发送的。这意味着任何链上的任何核心合约都可以验证 VAA 的真实性。此外，如果 VAA 需要被送达到特定的目的地，那么确保其正确传递的责任则由中继器承担。
{% endhint %}

### Signatures

VAA的 [body](vaa.md#body) 被`keccak256`**两次**哈希处理以产生签名的摘要消息。

```rust
// hash the bytes of the body twice
digest = keccak256(keccak256(body))
// sign the result 
signature = ecdsa_sign(digest, key)
```

{% hint style="warning" %}
不同的 ECDSA 签名验证实现可能会对传递的消息进行`keccak256`哈希，因此必须小心传递正确的参数。

例如，[Solana secp256k1](https://docs.solana.com/developing/runtime-facilities/programs#secp256k1-program) 将对传递的消息进行哈希。在这种情况下，消息的参数应该是主体的**单个哈希**，而不是**两次哈希过**的主体。
{% endhint %}

### Payload 类型

在 wormhole 上构建的不同应用可能会为附加到 VAA 的 payload 指定一种格式。这个 payload 提供了目标链和合约所需的信息，以便它可以采取某些行动（例如，向接收地址铸造代币）。

#### Token 转账

代币通过锁定/铸造和销毁/解锁机制从一条链转移到另一条链。虽然许多桥都基于这个基本前提进行工作，但是这种实现是通过依赖Wormhole 提供的通用消息传递协议来实现的，以支持将锁定和销毁事件从一条链路由到另一条链。这使得 Wormhole 的代币桥完全不依赖于任何特定的链。只要我们想要转移至的那条链上存在一个 wormhole 合约，就可以快速地将一个实现方案纳入网络中。由于 Wormhole 具有通用消息传递性质，发出消息的程序无需了解其他任何链的实施细节。

为了将代币从 A 转移到 B，我们必须在 A 上锁定代币并在 B 上铸造它们。重要的是，在 B 上可以进行铸造之前，必须证明 A 上的代币已经被锁定。为了促进此过程，A 首先会锁定代币，并发出一条表明已完成锁定操作的信息。该信息具有以下结构，并被称为转账信息:

```rust
u8      payload_id = 1          // Token Transfer
u256    amount                  // Amount of tokens being transferred.
u8[32]  token_address           // Address on the source chain.
u16     token_chain             // Numeric ID for the source chain.
u8[32]  to                      // Address on the destination chain.
u16     to_chain                // Numeric ID for the destination chain.
u256    fee                     // Portion of amount paid to a relayer.
```

此结构包含接收链了解锁定事件所需的所有内容。一旦链 B 收到此 payload ，它就可以铸造相应的资产。
请注意，链 B 无法确定发送方的代币是如何锁定的。它们可能已被铸币厂烧毁或锁定在托管账户中。一旦有足够数量的 guardians 证明其存在，协议就会简单地转发该事件。

#### 证明

上面的转账事件缺少一个重要细节。虽然 Chain B 上的程序可以信任该消息来通知它代币锁定事件，但它无法知道被锁定的代币实际上是什么。对于大多数用户来说，地址本身是一个毫无意义的值。为了解决这个问题，Token Bridge 支持 _token attestation_。
对于代币的证明，链 A 发出一条包含有关代币的元数据的消息，链 B 可以使用它来保存代币地址的名称、符号和十进制精度。
该操作的消息格式如下:

```rust
u8         payload_id = 2   // Attestation
[32]byte   token_address    // Address of the originating token contract
u16        token_chain      // Chain ID of the originating token 
u8         decimals         // Number of decimals this token should have
[32]byte   symbol           // Short name of asset
[32]byte   name             // Full name of asset
```

证明使用固定长度的字节数组来编码 UTF8 代币名称和 symbol 数据。

{% hint style="warning" %}
因为字节数组是固定长度的，所包含的数据可能会截断多字节 Unicode 字符。
{% endhint %}

当 _sending_ 一个证明 VAA 时, 我们建议发送不会截断字符的最长的 UTF8 前缀，然后用 0 字节向右填充。

当 _parsing_ 一个证明 VAA 时, 我们建议修剪所有尾随的 0 字节，然后通过任意有损算法将剩余部分转换为 UTF8 。

{% hint style="info" %}
请注意，不同的链上系统可能有不同的 VAA 解析器，如果字符串很长或包含无效的 UTF8，可能会导致不同链上的 names/symbols 不同。
{% endhint %}

代币桥的一个重要细节是，在代币转移之前需要进行证明。这是因为如果不知道代币的小数精度，链 B 就不可能在处理转移时正确地铸造正确数量的代币。



#### Token + 消息

{% hint style="info" %}
这种 VAA 类型也被称为 _payload3_ message 或者 _Contract Controlled Transfer_.
{% endhint %}

Token + 消息 数据结构与只有 Token 的数据结构相同，但增加了一个包含任意字节的`payload`字段。应用程序可以在此任意字节字段中包含额外的数据以通知某些特定于应用程序的行为。

```rust
u8      payload_id = 3 // Token Transfer with Message 
u256    amount         // Amount of tokens being transferred.
u8[32]  token_address  // Address on the source chain.
u16     token_chain    // Numeric ID for the source chain.
u8[32]  to             // Address on the destination chain.
u16     to_chain       // Numeric ID for the destination chain.
u256    fee            // Portion of amount paid to a relayer.
[]byte  payload        // Message, arbitrary bytes, app specific
```

#### 治理

治理 VAA 没有像上述格式那样的“payload_id”字段，它用于触发已部署合约中的某些操作（例如升级）。

**Action 结构**

治理消息包含预定义的操作，这些操作可以针对当前部署在链上的各种 wormhole 模块。该结构包含以下字段：

```rust
u8[32]  module // Contains a right-aligned module identifier
u8      action // Predefined governance action to execute
u16     chain  // Chain the action is targeting, 0 = all chains
...     args   // Arguments to the action
```

以下是一个示例消息，其中包含了触发对 solana 核心合约进行代码升级的治理行动。这里的模块字段是 ASCII "Core" 的右对齐编码，表示为32字节的十六进制字符串。

```rust
module:       0x00000000000000000000000000000000000000000000000000000000436f7265
action:       1
chain:        1
new_contract: 0x3485672937589571623749593761923748845625222819374462348283888283
```

**Actions**

每个 numeric action 的含义都在 wormhole 设计文档中预先定义和记录。对于每个应用程序，相关定义可以通过以下链接找到：

* 核心治理的 actions [这里](https://github.com/wormhole-foundation/wormhole/tree/main/whitepapers/0002\_governance\_messaging.md).
* 代币桥的治理的 actions [这里](https://github.com/wormhole-foundation/wormhole/tree/main/whitepapers/0003\_token\_bridge.md).
* NFT 桥的治理的 actions [这里](https://github.com/wormhole-foundation/wormhole/tree/main/whitepapers/0006\_nft\_bridge.md).

## 消息的生命周期

{% hint style="warning" %}
注意：任何人都可以将 VAA 提交到目标链。Guardians 通常不会执行此步骤以避免交易费用。相反，在 wormhole 之上构建的应用程序可以通过 guardian RPC 获取 VAA，并在单独的流程中进行提交。
{% endhint %}

现在，我们定义了这些概念，可以说明在两个链之间传递消息的完整流程。以下阶段演示了虫洞网络为路由消息而执行的每个处理阶段。

*   **1: 链 A 上运行的合约发出一条消息。**

    任何合约都可以发出消息，Guardians 被编程去观察所有链上的这些事件。在这里，Guardians 被表示为一个实体以简化流程，但消息的观察必须由 19 个 guardians 中的每一个单独执行。
*   **2: 签名已被聚合。**

    Guardians 独立观察并签署消息。一旦有足够的 guardians 签署了该消息，这些签名就会与消息和元数据结合起来生成一个 VAA。
*   **3: VAA 已提交至目标链。**

    VAA 充当了 guardians 共同证明消息 payload 存在的证明，为了完成最后一步，VAA 本身被提交（或 [被中继](relayer.md)）到目标链以由接收合约进行处理。
