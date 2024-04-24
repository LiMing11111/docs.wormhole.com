# 使用 Connect SDK 进行 USDC 转账

Connect SDK 通过 Circle 的跨链转账协议（CCTP）支持快速、低成本的本地 USDC 跨链桥接功能。使用 [Connect SDK](../../../reference/sdk-docs/) 通过 [CCTP](./) 转移本地 USDC 到其他链的操作非常类似于使用 SDK 进行标准的代币转移。

### 安装

首先安装 SDK 包。

```sh
npm install @wormhole-foundation/sdk
```

### 使用方法

要使用 CCTP 桥接功能，必须导入并注册平台。

```ts
import { Wormhole } from "@wormhole-foundation/sdk";
import evm from "@wormhole-foundation/sdk/evm";

// ...

const wh = await wormhole("Testnet", [evm]);
```

### 手动转账

```ts
  const srcAddress = Wormhole.chainAddress("Ethereum", "0xdeadbeef...") 
  const dstAddress = Wormhole.chainAddress("Avalanche", "0xbeefdead...") 

  const xfer = await wh.circleTransfer(
    1_000_000n, // 
    srcAddress,
    dstAddress
  );
  console.log(xfer);
```

手动转账包含三个步骤：

1. 通过调用 `initiateTransfer` 并传递一个 [Signer](../../../reference/sdk-docs/#signers) 来签署交易，启动转账。

```ts
  console.log("Starting Transfer");
  const srcTxids = await xfer.initiateTransfer(src.signer);
  console.log(`Started Transfer: `, srcTxids);
```

2. 等待 Circle 证明变得可用，可选择性地传递参数。

```ts
  // See https://developers.circle.com/stablecoins/docs/required-block-confirmations for reasonable timeout settings
  // based on origin chain
  const timeout = 60 * 1000;

  console.log("Waiting for Attestation");
  const attestIds = await xfer.fetchAttestation(timeout);
  console.log(`Got Attestation: `, attestIds);
```

3. 通过调用 `completeTransfer`，并再次传递一个签名者用于目的链，来完成转账。

```ts
  console.log("Completing Transfer");
  const dstTxids = await xfer.completeTransfer(dst.signer);
  console.log(`Completed Transfer: `, dstTxids);
```

### 自动转账

使用自动中继功能更为简单，仅涉及启动转账。中继器基础设施将为您处理获取和交付证明的工作。

```ts
  const xfer = await wh.circleTransfer(
    amount,  
    srcAddress, 
    dstAddress,
    true,       // automatic transfer plz
    undefined,  // An arbitrary bytes payload if one is necessary
    0n,         // no native gas dropoff for this demo
  );
  console.log(xfer);

  console.log("Starting Transfer");
  const srcTxids = await xfer.initiateTransfer(src.signer);
  console.log(`Started Transfer: `, srcTxids);

```

### 完成部分转账

在手动转账启动但未完成的情况下，可以仅通过源链和交易哈希重新构建转账对象。

这在用户在完成转账前终止会话，或者用于调试时特别有用。

```ts
  const timeout = 60 * 1000

  // Rebuild the transfer from the source txid
  const xfer = await CircleTransfer.from(wh, {
    chain: "Avalanche",
    txid: "0x6b6d5f101a32aa6d2f7bf0bf14d72bfbf76a640e1b2fdbbeeac5b82069cda4dd",
  }, timeout);

  const dstTxIds = await xfer.completeTransfer(signer);
  console.log("Completed transfer: ", dstTxIds);
```

一个示例的完整源代码可以在[这里](https://github.com/wormhole-foundation/wormhole-sdk-ts/blob/develop/examples/src/cctp.ts)获取。
