# 简单中继器

## 中继器引擎

中继器引擎是一个旨在为自定义中继器提供结构和起点的包。

使用中继器引擎，开发人员可以编写特定的逻辑进行过滤，只接收他们关心的信息。

一旦收到 wormhole 消息，开发人员可以应用附加的逻辑来解析自定义 payloads 或将 VAA 提交到一个或多个目标链。

要使用中继器引擎，开发人员可以使用惯用的 express/koa 中间件启发的 api，来为自己的应用程序指定如何中继wormhole 信息，然后让库处理所有细节！

在此处查看[快速开始](./#quick-start)示例，或查看更高级的中继器应用程序，请参阅[高级示例](advanced-example.md)。

## 快速开始

该示例的源代码可在[此处](https://github.com/wormhole-foundation/relayer-engine/blob/main/examples/simple/src/app.ts)获取

### 安装包

首先，使用你最喜欢的包管理器安装 `relayer-engine` 包

```sh
npm i @wormhole-foundation/relayer-engine
```

### 启动后台进程

> 注意：这些进程_必须_ 被运行，下面的中继器应用程序才能正常工作

接下来，我们必须启动 Spy来监听守护者网络上发布的可用 VAAs 以及持久层，在本例中我们使用 Redis。

有关 Spy 的更多详细信息，请参阅[文档](https://docs.wormhole.com/wormhole/explore-wormhole/spy)

#### Wormhole 网络 Spy

为了让我们的中继器应用接收消息，必须运行一个本地 Spy 来监视守护者网络。我们的中继器应用将接收来自此 Spy 的更新内容。

<details>

<summary>Testnet Spy</summary>

```bash
docker run --platform=linux/amd64 \
-p 7073:7073 \
--entrypoint /guardiand ghcr.io/wormhole-foundation/guardiand:latest \
spy \
--nodeKey /node.key \
--spyRPC "[::]:7073" \
--env testnet
```

</details>

<details>

<summary>Mainnet Spy</summary>

```bash
docker run --platform=linux/amd64 \
-p 7073:7073 \
--entrypoint /guardiand ghcr.io/wormhole-foundation/guardiand:latest \
spy \
--nodeKey /node.key \
--spyRPC "[::]:7073" \
--env mainnet
```

</details>

#### Redis 持久性

> 注意：虽然我们在这里使用 Redis，但也可以通过实现相应的[接口](https://github.com/wormhole-foundation/relayer-engine/blob/main/relayer/storage/redis-storage.ts)，将持久层换成其他数据库。

Redis 实例也必须可用，用于持久化存储从 Spy 获取 VAAs 的工作数据。

```bash
docker run --rm -p 6379:6379 --name redis-docker -d redis
```

### 简单中继器代码示例

在下面的示例中，我们将：

1. 设置一个 StandardRelayerApp，为我们的中继器传递配置选项
2. 添加一个过滤器，只捕获我们的应用程序所关心的信息，并在获取 VAA 后通过回调对其进行处理
3. 启动中继器应用程序

```ts
import {
  Environment,
  StandardRelayerApp,
  StandardRelayerContext,
} from "@wormhole-foundation/relayer-engine";
import { CHAIN_ID_SOLANA } from "@certusone/wormhole-sdk";

(async function main() {
  // initialize relayer engine app, pass relevant config options
  const app = new StandardRelayerApp<StandardRelayerContext>(
    Environment.TESTNET,
    // other app specific config options can be set here for things
    // like retries, logger, or redis connection settings.
    {
      name: "ExampleRelayer",
    },
  );

  // add a filter with a callback that will be
  // invoked on finding a VAA that matches the filter
  app.chain(CHAIN_ID_SOLANA).address(
    // emitter address on Solana
    "DZnkkTmCiFWfYTfT41X3Rd1kDgozqzxWaHqsw6W4x2oe",
    // callback function to invoke on new message
    async (ctx, next) => {
      const vaa = ctx.vaa;
      const hash = ctx.sourceTxHash;
      console.log(
        `Got a VAA with sequence: ${vaa.sequence} from with txhash: ${hash}`,
      );
    },
  );

  // add and configure any other middleware ..

  // start app, blocks until unrecoverable error or process is stopped
  await app.listen();
})();
```

#### 解释

第一行有意义的代码实例化了 `StandardRelayerApp`，它是 `RelayerApp` 的子类，具有通用默认值。

```ts
export class StandardRelayerApp<
  ContextT extends StandardRelayerContext = StandardRelayerContext,
> extends RelayerApp<ContextT> {
  // ...
  constructor(env: Environment, opts: StandardRelayerAppOpts) {
```

我们在 `StandardRelayerAppOpts` 中传递的唯一字段是 name，用于帮助识别日志信息并在 Redis 中保留命名空间。

<details>

<summary>Other `StandardRelayerAppOpts` options</summary>

```ts
  wormholeRpcs?: string[];  // List of URLs from which to query missed VAAs
  concurrency?: number;     // How many concurrent requests to make for workflows
  spyEndpoint?: string;     // The hostname and port of our Spy
  logger?: Logger;          // A custom Logger
  privateKeys?: Partial<{ [k in ChainId]: any[]; }>; // A set of keys that can be used to sign and send transactions
  tokensByChain?: TokensByChain;    // The token list we care about
  workflows?: { retries: number; }; // How many times to retry a given workflow
  providers?: ProvidersOpts;        // Configuration for the default providers
  fetchSourceTxhash?: boolean;      // whether or not to get the original transaction id/hash
  // Redis config
  redisClusterEndpoints?: ClusterNode[];
  redisCluster?: ClusterOptions;
  redis?: RedisOptions;
```

</details>

示例中的下一个有意义的行添加了一个过滤器中间件组件。该中间件将使中继器应用程序向 Spy 请求订阅任何符合条件的 VAAs，并调用 VAA 的回调函数。

如果希望程序订阅多个链和地址，可以多次调用同一方法或使用 `multiple` helper。

```ts
app.multiple(
  {
    [CHAIN_ID_SOLANA]: "DZnkkTmCiFWfYTfT41X3Rd1kDgozqzxWaHqsw6W4x2oe"
    [CHAIN_ID_ETH]: ["0xabc1230000000...","0xdef456000....."]
  },
  myCallback
);
```

简单示例中的最后一行运行 `await app.listen()`，这将启动中继引擎。一旦启动，中继引擎就会向 spy 发出订阅请求，并开始任何其他工作流（追踪错过的VAA）。

这将一直运行，直到进程被终止或遇到不可恢复的错误。如果您想正常关闭中继器，请调用 `app.stop()`。

### 高级示例

有关详细介绍其他中间件和更复杂的配置和操作（包括内置 UI）的更高级示例，请参阅[高级教程](advanced-example.md)

{% hint style="info" %}
### Wormhole 集成完成了吗？

请告诉我们，这样我们就可以将你的项目列入我们的生态目录，并将你介绍给我们的全球多链社区！

[立即联系！](https://forms.clickup.com/45049775/f/1aytxf-10244/JKYWRUQ70AUI99F32Q)
{% endhint %}
