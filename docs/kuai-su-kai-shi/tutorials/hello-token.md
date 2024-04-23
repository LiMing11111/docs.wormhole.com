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
2.  Off chain: [Fetch the VAA](https://docs.wormhole.com/wormhole/reference/api-docs/swagger#v1-signed\_vaa-chain\_id-emitter-seq) using the Wormhole Chain ID, Emitter address (TokenBridge address) and sequence number from the `LogMessage` event.

    > This is the VAA that contains the token details with signatures from the Guardians
3.  On the Receiving side: Calls the TokenBridge `createWrapped` function with the VAA from the previous step

    > This allows the TokenBridge to create a wrapped version of the token we're sending so that it may mint the tokens to the receiver.

Once this is done, the TokenBridge on the receiving side can successfully mint the token sent.

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
* We must encode the recipient address into a payload `bytes memory payload = abi.encode(recipient);`
* We must ensure the correct amount of `msg.value` was passed in to send the token and payload.
  * The cost to send a token is provided by the value returned by `wormhole.messageFee()` Currently this is 0 but _may_ change in the future, so don't assume it will always be 0.
  * The cost to request a relay depends on the gas amount and receiver value you will need. `(deliveryCost,) = wormholeRelayer.quoteEVMDeliveryPrice(targetChain, 0, GAS_LIMIT);`

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

### Implement Receiving Function

Now, we'll implement the `TokenReceiver` abstract class - which is also included in the Wormhole Solidity SDK

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

After we call `sendTokenWithPayloadToEvm` on the source chain, the message goes through the standard Wormhole message lifecycle. Once a VAA is available, the delivery provider will call `receivePayloadAndTokens` on the target chain and target address specified, with the appropriate inputs.

The arguments `payload`, `sourceAddress`, `sourceChain`, and `deliveryHash` are all the same as on the normal `receiveWormholeMessages` endpoint.

Let's delve into the fields that are provided to us in the `TokenReceived` struct:

* **tokenHomeAddress** The same as the `token` field in the call to `sendTokenWithPayloadToEvm`, as that is the original address of the token unless the original token sent is a wormhole-wrapped token. In the case a wrapped token is sent, this will be the address of the original version of the token (on it’s native chain) in [wormhole address format](https://docs.wormhole.com/wormhole/reference/environments/evm#addresses) - i.e. left-padded with 12 zeros
* **tokenHomeChain** The chain (in wormhole chain ID format) corresponding to the home address above - this will be the source chain, unless if the original token sent is a wormhole-wrapped asset, in which case it will be the chain of the unwrapped version of the token.
* **tokenAddress** This is the address of the IERC20 token on this chain (the target chain) that has been transferred to this contract. If tokenHomeChain == this chain, this will be the same as tokenHomeAddress; otherwise, it will be the wormhole-wrapped version of the token sent.
* **amount** This is the amount of the token that has been sent to you - the units being the same as the original token. Note that since TokenBridge only sends with 8 decimals of precision, if your token had 18 decimals, this will be the ‘amount’ you sent, rounded down to the nearest multiple of 10^10.
* **amountNormalized** This is the amount of token divided by (1 if decimals ≤ 8, else 10^(decimals - 8))

Since all we intend to do is send the received token to the recipient, our fields of interest are **payload** (containing recipient), **receivedTokens\[0].tokenAddress** (token we received), and **receivedTokens\[0].amount** (amount of token we received and that we must send)

We can complete the implementation as follows:

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

> Note: In this case, we don't need to prevent duplicate deliveries using the delivery hash, because TokenBridge already provides a form of duplicate prevention when redeeming sent tokens

And voila! We have a [complete working example](https://github.com/wormhole-foundation/hello-token/blob/main/src/HelloToken.sol) of a cross-chain application that uses TokenBridge to send and receive tokens!

Try [cloning and running HelloToken](https://github.com/wormhole-foundation/hello-token/tree/main#readme) to see this example work for yourself!

## How do these Solidity Helpers Work?

Let’s walk through the details of `sendTokenWithPayloadToEvm` and `receivePayloadAndTokens` to see how they make use of the IWormholeRelayer interface and IWormholeReceiver interface to send and receive tokens.

### Sending a Token

To send a token, we make use of the EVM TokenBridge contract, specifically the `transferTokensWithPayload` method ([implementation](https://github.com/wormhole-foundation/wormhole/blob/main/ethereum/contracts/bridge/Bridge.sol#L191))

> Note: We leave the `payload` field here blank because we are using the `payload` field on the IWormholeRelayer interface instead

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

TokenBridge implements this function by publishing a wormhole message to the blockchain logs that indicates that `amount` of the `token` was sent (with the intended address being `recipient` on `recipientChain`). TokenBridge then returns the sequence number of this published wormhole message.

The `transferTokens` function in the Wormhole Solidity SDK makes use of this TokenBridge endpoint by

* approving the TokenBridge to spend `amount` of our ERC20 `token`
* calling `transferTokensWithPayload` with the appropriate inputs
* returning a `VaaKey` struct containing information about the published wormhole message for the token transfer

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

Now, it is our task to get the signed VAA corresponding to this published token bridge wormhole message to be delivered to our target chain HelloToken contract. To do this, we make use of the [sendVaasToEvm](https://github.com/wormhole-foundation/wormhole/blob/main/ethereum/contracts/interfaces/relayer/IWormholeRelayer.sol#L149) endpoint in the IWormholeRelayer interface.

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

This allows us to specify existing wormhole message(s) and get the signed VAA(s) corresponding to those messages delivered to the targetAddress (in the `additionalVaas` field of `receiveWormholeMessages`).

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

> Note: If you wish to send multiple different tokens along with the payload, the `sendTokenWithPayloadToEvm` helper as currently implemented will not help (as it sends only one token). However, you can still call `transferToken` twice and request delivery of both of those TokenBridge wormhole messages by providing two `VaaKey` structs in the `vaaKeys` array. See an example of HelloToken with more than one token [here](https://github.com/wormhole-foundation/hello-token/blob/main/src/example-extensions/HelloMultipleTokens.sol).

### Receiving a Token

We know that our `sendVaasToEvm` call will cause `receiveWormholeMessages` on `targetAddress` to be called with

* The payload as the encoded `recipient` address
* The `additionalVaas` field being an array of length 1, with the first element being the signed VAA corresponding to our token bridge transfer

Crucially, we don't have the transferred tokens yet! There are a few things that we need to do before gaining access to these tokens.

1.  We parse the signed VAA, and check that

    * The emitterAddress of the VAA is a valid token bridge - i.e. the message was published by one of the TokenBridge contracts
    * The transfer was sent to this address

    > note: this step isn’t strictly necessary because the call to `completeTransferWithPayload` would fail if these were not true\*\*
2. We call `tokenBridge.completeTransferWithPayload`, passing the VAA - this completes the transfer of the tokens and causes us to receive the (potentially wormhole-wrapped) transferred token
3. We return a `TokenReceived` struct containing useful information about the transfer
4. We call `receivePayloadAndTokens` with the appropriate inputs

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

See the full implementation of the Wormhole Relayer SDK helpers [here](https://github.com/wormhole-foundation/wormhole-solidity-sdk/blob/main/src/WormholeRelayerSDK.sol)

Also, see a version of HelloToken implemented without any Wormhole Relayer SDK helpers [here](https://github.com/wormhole-foundation/hello-token/blob/main/src/example-extensions/HelloTokenWithoutSDK.sol)

as well as a version of HelloToken where native currency is deposited [here](https://github.com/wormhole-foundation/hello-token/blob/main/src/example-extensions/HelloTokenNative.sol)

{% hint style="info" %}
### Wormhole integration complete?

Let us know so we can list your project in our ecosystem directory and introduce you to our global, multichain community!

[Reach out now!](https://forms.clickup.com/45049775/f/1aytxf-10244/JKYWRUQ70AUI99F32Q)
{% endhint %}
