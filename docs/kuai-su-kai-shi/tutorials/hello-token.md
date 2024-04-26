# Hello Token

本教程包含一个 [solidity 合约](https://github.com/wormhole-foundation/hello-token/blob/main/src/HelloToken.sol)，它可以部署到许多 EVM 链上，形成一个功能完整的跨链应用程序，用户可以通过一个合约请求将 tokens 发送到不同链上的地址。

下面是一个[跨链借贷应用](https://github.com/wormhole-foundation/cross-chain-borrow-lend)的示例，其中使用了本教程中涉及的主题！

## 概要

此[仓库](https://github.com/wormhole-foundation/hello-token/)包含以下内容：

* Solidity 代码示例
* Forge 本地测试配置示例
* Testnet 部署脚本
* Testnet 测试配置示例

### 环境配置

* Node 16.14.1 或更高版本，npm 8.5.0 或更高版本： [https://docs.npmjs.com/downloading-and-installing-node-js-and-npm](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm)
* forge 0.2.0 或更高版本：[https://book.getfoundry.sh/getting-started/installation](https://book.getfoundry.sh/getting-started/installation)

### 本地测试

克隆该仓库，并进入，然后构建并运行单元测试：

```bash
git clone https://github.com/wormhole-foundation/hello-token.git
cd hello-token
npm run build
forge test
```

预期输出为

```bash
Running 1 test for test/HelloToken.t.sol:HelloTokenTest
[PASS] testCrossChainDeposit() (gas: 1338038)
Test result: ok. 1 passed; 0 failed; finished in 5.64s
```

### 部署到 Testnet

你将需要一个至少有 0.05 Testnet AVAX 和 0.01 Testnet CELO 的钱包。

* [在此处获取 testnet AVAX](https://core.app/tools/testnet-faucet/?token=C)
* [在此处获取 testnet CELO](https://faucet.celo.org/alfajores)

```bash
EVM_PRIVATE_KEY=your_wallet_private_key npm run deploy
```

### 在 Testnet 上测试

你将需要一个至少有 0.02 Testnet AVAX 的钱包。[在此处获取 testnet AVAX](https://core.app/tools/testnet-faucet/?token=C)

你还必须将合约部署到 testnet 上（如上节所述）。

要测试在 testnet 上发送和接收信息，请按如下方式执行测试：

```bash
EVM_PRIVATE_KEY=your_wallet_private_key npm run test
```

## 入门

让我们编写一个 [HelloToken 合约](https://github.com/wormhole-foundation/hello-token/blob/main/src/HelloToken.sol)，让用户可以将任意数量的 IERC20 token 发送到他们在另一条链上选择的地址。

### 有效 Tokens

在开始之前，需要注意的是，我们使用 Wormhole 的 **TokenBridge** 在链之间传输 tokens！

因此，要使用本例中的方法发送 token，必须将 token 认证到我们期望的目标区块链上的 Token Bridge 合约上。

在上面的测试中，当你运行 `npm run deploy` 时， 一个模拟 token 合约被部署并认证到目标链的 Token Bridge 合约上。

如果你希望自己为 TokenBridge 认证 token，你可以使用 [attestWorkflow](https://github.com/wormhole-foundation/hello-token/blob/main/ts-scripts/deploy-mock-tokens.ts#L52) 函数。

要检查 token 是否已被认证到 TokenBridge 上，可调用 TokenBridge 上的 `wrappedAsset(uint16 tokenChainId, bytes32 tokenAddress)` 函数，如果已被认证，则将返回此区块链上与给定 token（来自源区块链）相对应的 wrapped token 的地址，如果输入 token 尚未认证，则返回 0 地址。

<details>

<summary>attestWorkflow 的工作原理</summary>

The 'attestWorkflow' 函数的功能如下：

1.  在 Source 端: 使用我们要尝试发送的 token 调用 TokenBridge `attestToken` 函数。

    > 这将创建一个包含 token 详细信息的 payload，以便在接收端创建 token
2.  链下：使用Wormhole Chain ID、Emitter 地址（TokenBridge 地址）和来自 `LogMessage` 事件的序列号[获取 VAA](https://docs.wormhole.com/wormhole/reference/api-docs/swagger#v1-signed\_vaa-chain\_id-emitter-seq)。

    > 这是包含来自 Guardians 签名的 token 详细信息的 VAA
3.  在接收端：使用上一步的 VAA 调用 TokenBridge 的 `createWrapped` 函数。

    > 这允许 TokenBridge 为我们发送的 token 创建一个包装版本，以便它可以将 token 铸造给接收方。

一旦完成，接收端的 TokenBridge 就能成功地铸造发送的 token。

</details>

### Wormhole Solidity SDK

为了便于开发，我们将使用 [Wormhole Solidity SDK](https://github.com/wormhole-foundation/wormhole-solidity-sdk)。

在你自己的跨链应用程序中运行该 SDK：

```bash
forge install wormhole-foundation/wormhole-solidity-sdk
```

并将其导入到你的合约中：

```solidity
import "wormhole-solidity-sdk/WormholeRelayerSDK.sol";
```

该 SDK 提供了帮助工具，使 Wormhole 的跨链开发变得更容易，特别是为我们提供了 TokenSender 和 TokenReceiver 抽象类，它们具有使用 TokenBridge 发送和接收 tokens 的有用功能。

### 实现发送功能

让我们首先编写一个函数，将一定数量的 token 发送给目标链上的特定接收者。

```solidity
function sendCrossChainDeposit(
        uint16 targetChain, // A wormhole chain id
        address targetHelloToken, // address of HelloToken contract on targetChain
        address recipient,
        uint256 amount,
        address token
) public payable;
```

该函数的主体将向目标链上的 HelloToken 合约发送 token 和 payload 。对于我们的应用程序来说，payload 将包含 token 的预期接收者，这样目标链上的 HelloToken 合约就能把 token 发送给预期接收者。

> 注意：TokenBridge 只支持发送 IERC20 tokens，具体而言只支持最多 8 位小数的 token。因此，如果你的 IERC20 token 有 18 位小数，并且你发送的是一个 token 的 `amount`，那么你将收到四舍五入到最接近的 10^10 倍的 `amount`。

为了向 HelloToken 合约发送 token 和 payload，我们使用了 Wormhole Solidity SDK 的 `sendTokenWithPayloadToEvm` 帮助工具。

想要成功传送，需要做到以下几点：

* 调用 `sendCrossChainDeposit` 的用户（或合约）应**批准** `HelloToken` 合约使用用户 tokens 的 `amount`。在[此处](https://github.com/wormhole-foundation/hello-token/blob/main/test/HelloToken.t.sol#L37)查看如何在 forge 测试中完成此操作。
* 我们必须将用户的 token 中的 `amount` 转移到 `HelloToken` 源合约中`IERC20(token).transferFrom(msg.sender, address(this), amount);`
* 我们必须将接收方地址编码到 payload 中 `bytes memory payload = abi.encode(recipient);`
* 我们必须确保传递了正确数量的 `msg.value` 来发送 token 和 payload。
  * 发送 token 的成本由 `wormhole.messageFee()` 返回值决定，目前该值 0，但将来_可能_会发生变化，因此不要假设它总是为0。
  * 请求中继的成本取决于你所需的 gas 量和接收器值。`(deliveryCost,) = wormholeRelayer.quoteEVMDeliveryPrice(targetChain, 0, GAS_LIMIT);`

```solidity
function sendCrossChainDeposit(
        uint16 targetChain,
        address targetHelloToken,
        address recipient,
        uint256 amount,
        address token
) public payable {
    uint256 cost = quoteCrossChainDeposit(targetChain);
    require(msg.value == cost,
    "msg.value != quoteCrossChainDeposit(targetChain)");

    IERC20(token).transferFrom(msg.sender, address(this), amount);

    bytes memory payload = abi.encode(recipient);
    sendTokenWithPayloadToEvm(
       targetChain,
       targetHelloToken, // address (on targetChain) to send token and payload
       payload,
       0, // receiver value
       GAS_LIMIT,
       token, // address of IERC20 token contract
       amount
    );
}

function quoteCrossChainDeposit(uint16 targetChain)
public view returns (uint256 cost) {
    // Cost of delivering token and payload to targetChain
    uint256 deliveryCost;
    (deliveryCost,) =
    wormholeRelayer.quoteEVMDeliveryPrice(targetChain, 0, GAS_LIMIT);

    // Total cost: delivery cost +
    // cost of publishing the 'sending token' wormhole message
    cost = deliveryCost + wormhole.messageFee();
}
```

### 实现接收函数

现在，我们将实现 `TokenReceiver` 抽象类——它也包含在 Wormhole Solidity SDK 中

```solidity
struct TokenReceived {
    bytes32 tokenHomeAddress;
    uint16 tokenHomeChain;
    address tokenAddress;
    uint256 amount;
    uint256 amountNormalized;
}

function receivePayloadAndTokens(
        bytes memory payload,
        TokenReceived[] memory receivedTokens,
        bytes32 sourceAddress,
        uint16 sourceChain,
        bytes32 deliveryHash
) internal virtual {}
```

在源链上调用 `sendTokenWithPayloadToEvm` 后，信息将经历标准的 Wormhole 信息生命周期。一旦 VAA 可用，delivery provider 将在目标链和指定的目标地址上调用 `receivePayloadAndTokens`，并输入适当的内容。

参数 `payload`、`sourceAddress`、`sourceChain` 和 `deliveryHash` 都与普通的 `receiveWormholeMessages` 端点上的相同。

让我们深入了解一下 `TokenReceived` 结构中提供给我们的字段：

* **tokenHomeAddress** 与调用 `sendTokenWithPayloadToEvm` 时的 `token` 字段相同，因为这是 token 的原始地址，除非发送的原始 token 是 wormhole 包装的 token。如果发送的是包装的 token，则该地址将是原始版本 token（在其原生链）的地址，采用 [wormhole 地址格式](https://docs.wormhole.com/wormhole/reference/environments/evm#addresses)，即左侧填充 12 个 0。
* **tokenHomeChain** 与上述主地址相对应的链（wormhole 链 ID 格式）——这将是源链，除非发送的原始 token 是 wormhole 包装的资产，在这种情况下它将是未包装版本 token 的链。
* **tokenAddress** 这是该链上（目标链）的 IERC20 token 的地址，该 token 地址已转移到此合约。如果 tokenHomeChain == 该链，则地址将与 tokenHomeAddress 相同；否则，它将是发送的 token 的 wormhole 包装版本。
* **amount** 这是已发送给你的 token 数量——单位与原始 token 相同。请注意，由于 TokenBridge 只以 8 位小数的精度发送，如果你的 token 有 18 位小数，这将是你发送的“amount”，四舍五入到最接近的 10^10 的 倍数。
* **amountNormalized** 这是 token 数量除以 (1 if decimals ≤ 8, else 10^(decimals - 8))

由于我们要做的只是将收到的 token 发送给接收方，因此我们感兴趣的字段是 **payload**（包含接收方）、**receivedTokens\[0].tokenAddress**（我们收到的token）和 **receivedTokens\[0].amount**（我们收到并必须发送的 token 数量）。

我们可以按照以下方式完成实现：

```solidity
function receivePayloadAndTokens(
        bytes memory payload,
        TokenReceived[] memory receivedTokens,
        bytes32, // sourceAddress
        uint16,
        bytes32 // deliveryHash
) internal override onlyWormholeRelayer {
    require(receivedTokens.length == 1, "Expected 1 token transfers");

    address recipient = abi.decode(payload, (address));

    IERC20(receivedTokens[0].tokenAddress).transfer(recipient, receivedTokens[0].amount);
}
```

> 注意：在这种情况下，我们不需要使用传递 hash 来防止重复传递，因为 TokenBridge 在履行发送 token 时已经提供了一种预防重复的方式。

好了！我们就有了一个使用 TokenBridge 发送和接收 tokens 跨链应用程序的[完整运行示例](https://github.com/wormhole-foundation/hello-token/blob/main/src/HelloToken.sol)！

尝试[克隆并运行 HelloToken](https://github.com/wormhole-foundation/hello-token/tree/main#readme) 来亲自看看这个示例是如何运行的！

## 这些 Solidity Helpers 如何工作？

让我们来了解一下 `sendTokenWithPayloadToEvm` 和 `receivePayloadAndTokens` 的细节，看看它们时如何利用 IWormholeRelayer 接口和 IWormholeReceiver 接口来发送和接收 tokens 的。

### 发送 Token

为了发送 token，我们使用 EVM TokenBridge 合约，特别是 `transferTokensWithPayload` 方法（[实现方式](https://github.com/wormhole-foundation/wormhole/blob/main/ethereum/contracts/bridge/Bridge.sol#L191)）

> 注意：我们将此处的 `payload` 字段留空，因为我们使用的是 IWormholeRelayer 接口上的 `payload` 字段。

```solidity
    /*
     *  @notice Send ERC20 token through portal.
     *
     *  @dev This type of transfer is called a "contract-controlled transfer".
     *  There are three differences from a regular token transfer:
     *  1) Additional arbitrary payload can be attached to the message
     *  2) Only the recipient (typically a contract) can redeem the transaction
     *  3) The sender's address (msg.sender) is also included in the transaction payload
     *
     *  With these three additional components, xDapps can implement cross-chain
     *  composable interactions.
     */
    function transferTokensWithPayload(
        address token,
        uint256 amount,
        uint16 recipientChain,
        bytes32 recipient,
        uint32 nonce,
        bytes memory payload
    ) public payable nonReentrant returns (uint64 sequence)
```

TokenBridge 通过向区块链日志发布 wormhole 信息来实现此功能，该信息表明`token` 的 `amount` 已被发送（目标地址为 `recipientChain` 上的 `recipient` ）。然后，TokenBridge 返回已发已发布 wormhole 信息的序列号。

Wormhole Solidity SDK 中的 `transferTokens` 函数通过以下方式使用该 TokenBridge 端点：

* 批准 TokenBridge 使用我们的 ERC20 `token` 的 `amount`
* 通过适当的输入调用 `transferTokensWithPayload`
* 返回一个 `VaaKey` 结构体，其中包含已发布的 token 传输的 wormhole 信息。

```solidity
function transferTokens(
        address token,
        uint256 amount,
        uint16 targetChain,
        address targetAddress
) internal returns (VaaKey memory) {
    IERC20(token).approve(address(tokenBridge), amount);

    uint64 sequence = tokenBridge.transferTokensWithPayload
    {value: wormhole.messageFee()}(
       token, amount, targetChain, toWormholeFormat(targetAddress), 0, bytes("")
    );

    return VaaKey({
        emitterAddress: toWormholeFormat(address(tokenBridge)),
        chainId: wormhole.chainId(),
        sequence: sequence
    });
}
```

现在，我们的任务是获取与发布的 token 桥 wormhole 信息相对应的已签名 VAA， 并将其传递到我们的目标链 HelloToken 合约。为此，我们使用 IWormholeRelayer 接口中的 [sendVaasToEvm](https://github.com/wormhole-foundation/wormhole/blob/main/ethereum/contracts/interfaces/relayer/IWormholeRelayer.sol#L149) 端点。

```solidity
function sendVaasToEvm(
        uint16 targetChain,
        address targetAddress,
        bytes memory payload,
        uint256 receiverValue,
        uint256 gasLimit,
        VaaKey[] memory vaaKeys
) external payable returns (uint64 sequence);
```

这样，我们就可以指定现有的 wormhole 信息，并获得与传递到 targetAddress（在 `receiveWormholeMessages` 的 `additionalVaas` 字段中）的信息相对应的签名 VAA(s)。

```solidity
function sendTokenWithPayloadToEvm(
        uint16 targetChain,
        address targetAddress,
        bytes memory payload,
        uint256 receiverValue,
        uint256 gasLimit,
        address token,
        uint256 amount
) internal returns (uint64) {
    VaaKey[] memory vaaKeys = new VaaKey[](1);
    vaaKeys[0] = transferTokens(token, amount, targetChain, targetAddress);

    (uint256 cost,) =
		wormholeRelayer.quoteEVMDeliveryPrice(targetChain, receiverValue, gasLimit);

    return wormholeRelayer.sendVaasToEvm{value: cost}(
        targetChain, targetAddress, payload, receiverValue, gasLimit, vaaKeys
    );
}
```

> 注意：如果你想连同 payload 一起发送多个不同的 tokens，目前实现的 `sendTokenWithPayloadToEvm` helper 将无能为力（因为它只发送一个 token）。不过，你仍然可以调用两次 `transferToken`，并通过在 `vaaKeys` 数组中提供两个 `VaaKey` 结构来请求传递这两个 TokenBridge wormhole 信息。请点击[此处](https://github.com/wormhole-foundation/hello-token/blob/main/src/example-extensions/HelloMultipleTokens.sol)查看带有多个 token 的 HelloToken 示例。

### 接收 Token

我们知道，我们的 `sendVaasToEvm` 调用会导致 `targetAddress` 上的 `receiveWormholeMessages` 被调用，调用条件是

* payload 作为编码的 `recipient` 地址
* `additionalVaas` 字段是一个长度为 1 的数组，第一个元素是与我们的 token 桥传递相对应的签名 VAA。

关键是，我们还没有要传递的 tokens！在获得这些 tokens 之前，我们需要做几件事。

1.  我们解析已签名的 VAA，并检查

    * VAA 的 emitterAddress 是一个有效的 token 桥，即信息由其中一个 TokenBridge 合约发布
    * transfer 已发送到该地址

    > 注意：严格来说，这一步并非必要的，因为如果这些信息不为真，调用 `completeTransferWithPayload` 就会失败\*\*
2. 我们调用 `tokenBridge.completeTransferWithPayload`，同时传递 VAA——这样就完成了 tokens 的传输，并使我们收到（可能被 wormhole 包装的）传输 token
3. 我们返回一个 `TokenReceived` 结构，其中包含有关传输的有用信息
4. 我们使用适当的输入调用 `receivePayloadAndTokens`

```solidity
function receiveWormholeMessages(
        bytes memory payload,
        bytes[] memory additionalVaas,
        bytes32 sourceAddress,
        uint16 sourceChain,
        bytes32 deliveryHash
    ) external payable {
    TokenReceived[] memory receivedTokens =
    new TokenReceived[](additionalVaas.length);

    for (uint256 i = 0; i < additionalVaas.length; ++i) {
        IWormhole.VM memory parsed = wormhole.parseVM(additionalVaas[i]);
        require(
            parsed.emitterAddress == tokenBridge.bridgeContracts(parsed.emitterChainId), "Not a Token Bridge VAA"
        );
        ITokenBridge.TransferWithPayload memory transfer = tokenBridge.parseTransferWithPayload(parsed.payload);
        require(
            transfer.to == toWormholeFormat(address(this)) && transfer.toChain == wormhole.chainId(),
            "Token was not sent to this address"
        );

        tokenBridge.completeTransferWithPayload(additionalVaas[i]);

        address thisChainTokenAddress = getTokenAddressOnThisChain(transfer.tokenChain, transfer.tokenAddress);
        uint8 decimals = getDecimals(thisChainTokenAddress);
        uint256 denormalizedAmount = transfer.amount;
        if (decimals > 8) denormalizedAmount *= uint256(10) ** (decimals - 8);

        receivedTokens[i] = TokenReceived({
            tokenHomeAddress: transfer.tokenAddress,
            tokenHomeChain: transfer.tokenChain,
            tokenAddress: thisChainTokenAddress,
            amount: denormalizedAmount,
            amountNormalized: transfer.amount
        });
    }

    // call into overriden method
    receivePayloadAndTokens(payload, receivedTokens, sourceAddress, sourceChain, deliveryHash);
}
```

点击[此处](https://github.com/wormhole-foundation/wormhole-solidity-sdk/blob/main/src/WormholeRelayerSDK.sol)查看 Wormhole Relayer SDK helpers 的完整实现

另外，请点击[此处](https://github.com/wormhole-foundation/hello-token/blob/main/src/example-extensions/HelloTokenWithoutSDK.sol)查看不使用任何 Wormhole Relayer SDK helpers 的情况下实现 HelloToken 的版本。

同时，还有一个版本的 HelloToken，在[这里](https://github.com/wormhole-foundation/hello-token/blob/main/src/example-extensions/HelloTokenNative.sol)存放了原生代币。

{% hint style="info" %}
### Wormhole 集成完成了吗？

请告诉我们，这样我们就可以将你的项目列入我们的生态目录，并将你介绍给我们的全球多链社区！

[立即联系！](https://forms.clickup.com/45049775/f/1aytxf-10244/JKYWRUQ70AUI99F32Q)
{% endhint %}
