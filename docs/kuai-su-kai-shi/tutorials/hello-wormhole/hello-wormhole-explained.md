# 解释 Hello Wormhole

在第 1 部分（ [HelloWormhole](./)）中，我们编写了一个功能完整的跨链应用程序， 允许用户从一个合约中请求，在不同链上的另一个合约中发出 `GreetingReceived` 事件。

为此，我们使用了 **Wormhole Relayer** 合约（[完整接口](https://github.com/wormhole-foundation/wormhole/blob/main/ethereum/contracts/interfaces/relayer/IWormholeRelayer.sol)，[实现方式](https://github.com/wormhole-foundation/wormhole/blob/main/ethereum/contracts/relayer/wormholeRelayer/WormholeRelayer.sol)）。

具体来说，我们使用了以下函数：

```solidity

    function sendPayloadToEvm(
        uint16 targetChain,
        address targetAddress,
        bytes memory payload,
        uint256 receiverValue,
        uint256 gasLimit
    ) external payable returns (uint64 sequence);

    function quoteEVMDeliveryPrice(
        uint16 targetChain,
        uint256 receiverValue,
        uint256 gasLimit
    ) external view returns (uint256 nativePriceQuote, uint256);
```

该 **Wormhole Relayer** 合约同事部署在 Mainnet 和 Testnets 上。

### Testnet

* Chain ID 4: BSC Testnet `0x80aC94316391752A193C1c47E27D382b507c93F3`
* Chain ID 5: Polygon Testnet `0x0591C25ebd0580E0d4F27A82Fc2e24E7489CB5e0`
* Chain ID 6: Avalanche Testnet (Fuji) `0xA3cF45939bD6260bcFe3D66bc73d60f19e49a8BB`
* Chain ID 14: Celo Testnet: `0x306B68267Deb7c5DfCDa3619E22E9Ca39C374f84`
* Chain ID 16: Moonbeam Testnet `0x0591C25ebd0580E0d4F27A82Fc2e24E7489CB5e0`

### Mainnet

在所有 Mainnet 部署中，合同都有相同的地址：`0x27428DD2d3DD32A4D7f7C497eAaa23130d894911`

它目前部署在：

* Chain ID 2: Ethereum
* Chain ID 4: Binance Smart Chain
* Chain ID 5: Polygon
* Chain ID 6: Avalanche
* Chain ID 10: Fantom
* Chain ID 13: Klaytn
* Chain ID 14: Celo
* Chain ID 16: Moonbeam
* Chain ID 23: Arbitrum
* Chain ID 24: Optimism

> 注意：输入的 `targetChain` 应为你所需链前面的数字；例如 2 代表 Ethereum， 4 代表 Binance Smart Chain， 6 代表 Avalanche，等等...

因此对于任何这些链，都可以调用已部署合约的 `sendPayloadToEvm`。

> 注意：必须使用特定 `msg.value` 调用 `sendPayloadToEvm` 方法，具体为 `(uint256 requiredMsgValue,) = quoteEVMDeliveryPrice(targetChain, receiverValue, gasLimit)`

调用此函数将导致在链 'targetChain' 上的地址 'targetAddress' 上的 **receiveWormholeMessages** 端点被调用，调用的 gas 限制为 'gasLimit'，msg.value 为 'receiverValue'，并且 payload 参数为 'payload'。&#x20;

## Wormhole Relayer 合约如何在不同的区块链上进行函数调用？

### 步骤 1

**Wormhole Relayer 合约发布 Delivery Instructions 并向支付 Delivery Provider**

`sendPayloadToEvm` 方法的工作原理是发布一条 wormhole 信息（只是一个包含字节的事件），其中包含有关如何执行交付的说明：目标链、目标地址、接收者值、gas limit、payload 及任何其他必要信息

`sendPayloadToEvm` 还会向 **delivery provider** 支付其 `msg.value`。

Delivery Providers 是无需许可的实体，可帮助支持 Wormhole Relayer 网络。如果未指定，你的支付请求将被分配给默认 delivery provider。每个 delivery provider 都可以为中继到具有特定 receiverValue 和 gasLimit 的特定链设定自己的定价。

[完整的 Wormhole Relayer 接口](https://github.com/wormhole-foundation/wormhole/blob/main/ethereum/contracts/interfaces/relayer/IWormholeRelayer.sol)提供了端点，你可以在其中指定要使用的  delivery provider

> **在我们的方案中，**
>
> * HelloWormhole 的用户调用 ‘sendCrossChainGreeting’，
> * 这将是调用 Wormhole Relayer 合约的 ‘sendPayloadToEvm’，
> * 将交付指令发布到区块链日志并向默认 delivery provider 支付

### 步骤 2

**Guardians 创建已签名的 VAA**

wormhole 协议的核心是发布区块链上的信息，然后由称为 [Guardians](https://docs.wormhole.com/wormhole/explore-wormhole/guardian) 的19个实体组成的的法定人数进行签名，形成已签名的 VAAs。

每个 Guardian 都会监控 wormhole 连接的区块链，并在看到的 Wormhole 事件上签名，从而形 [wormhole 的核心基元 ](https://docs.wormhole.com/wormhole/explore-wormhole/vaa)VAA。一旦 VAA 获得 19 个签名中的 13 个，它就被视为完全签名。

> 在我们的方案中，19 位 guardians 中有 13 位在交付说明上签名，以形成一份已签名的 VAA

### 步骤 3

**Delivery Provider 监控已签名的 VAAs，其中包含它已被分配的交付内容，并解析交付指令（链下！）**

Delivery Provider 很可能正在运行某种形式的 [Relayer Engine](https://github.com/wormhole-foundation/relayer-engine)，监控着 guardian 网络中的已签名的 VAAs，其中包含来自Wormhole Relayer 合约的 wormhole 信息，这些消息指示他们需要执行的交付指令。

> 交付指令甚至可以指示获取其他 VAAs（ [请参阅 sendVaasToEvm 端点](https://github.com/wormhole-foundation/wormhole/blob/main/ethereum/contracts/interfaces/relayer/IWormholeRelayer.sol#L119)！）， 此功能对于其他 Wormhole 协议，例如 TokenBridge 组合非常有用

Delivery Provider 解析看到的传递指令，并获取以下信息：

* 完整签名的 VAA， 其中包含交付说明（以解析）
* 要求交付的任何其他 VAAs
* 目标链
* 交付成本（需要提供 msg.value 以及可能需要支付的最大退款金额）

然后，使用已签名的交付指令 VAA 和 额外的 VAAs（以及 msg.value）作为输入，调用目标链上 Wormhole Relayer 合约的 `deliver` 端点。

> **在我们的方案中，**
>
> * 默认 delivery provider 查看交付说明 VAA，&#x20;
> * 解析链下的 VAA 以找出正确的目标链
> * 并将 VAA 提交给目标链上的 Wormhole Relayer 合约的 ‘deliver’ 端点

### 步骤 4

**Wormhole Relayer 合约接收交付的 VAA，确保 guardian 签名有效，并调用 `receiveWormholeMessages` 端点**

当 delivery provide 调用 Wormhole Relayer 合约上的交付端点时：&#x20;

* 确保交付 VAA 的签名有效
* 解析交付指令，找出targetAddress、payload、gasLimit、receiverValue，等
* **调用 'targetAddress' 的 'receiveWormholeMessages()' 端点**，并传入 payload，额外的数据源（例如额外的 VAAs、源链、源地址、交付 VAA 的哈希，可以用作唯一标识符），以及指定的 gasLimit 和 msg.value（receiverValue）。&#x20;

然后发出一个状态事件来指示此调用是否成功或失败（如果失败，则提供恢复字符串）。

> 要查看交付请求的状态，请使用 Wormhole Javascript SDK  中的 'getWormholeRelayerInfo' 函数——[请参见此处的方法](https://github.com/wormhole-foundation/hello-wormhole/blob/main/ts-scripts/getStatus.ts)。你可以在 HelloWormhole 中运行以下命令： `npm run status -- --tx TRANSACTION_HASH`

> **在我们的方案中红，（在目标链上）**
>
> * Wormhole Relayer 合约会验证交付 VAA 上的签名，
> * 然后解析 VAA，找出正确的目标地址（也就是我们在这条链上的 HelloWormhole 合约 ）、payload 和 gas limit
> * 并将 payload 提交给我们的 HelloWormhole 合约的 ‘receiveWormholeMessages’ 端点
> * ....然后我们在目标链上的 HelloWormhole 合约就完成了其余的工作！

{% hint style="info" %}
### Wormhole 集成完成了吗？

请告诉我们，这样我们就可以将你的项目列入我们的生态目录，并将你介绍给我们的全球多链社区！

[立即联系！](https://forms.clickup.com/45049775/f/1aytxf-10244/JKYWRUQ70AUI99F32Q)
{% endhint %}
