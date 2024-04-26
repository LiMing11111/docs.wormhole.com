# 高级中继器示例

本示例详细介绍了中继器应用程序更复杂的实现。有关简单示例，请参阅此[示例](./#simple-relayer-code-example)

本示例的源代码可在[此处](https://github.com/wormhole-foundation/relayer-engine/blob/main/examples/advanced/src/app.ts)获取

## 设置

> 注意：为了运行本教程中的 Spy 和 Redis，你必须安装 `docker`。

### 获取代码

克隆仓库， `cd` 进入该目录，然后安装依赖。

```sh
git clone https://github.com/wormhole-foundation/relayer-engine.git
cd relayer-engine/examples/advanced/
npm i
```

## 运行

### 启动后台服务

启动 Spy 来订阅 Guardian 网络上的各类信息。

```sh
npm run testnet-spy
```

在另一个 CLI 窗口中，启动 Redis。对于这个应用程序，Redis 被用作持久化层来跟踪我们已经看到的 VAAs。

```sh
npm run redis
```

### 启动中继器

一旦后台进程运行起来，我们就可以启动中继器。它将订阅来自 Spy 的相关消息并跟踪 VAAs，并在收到消息时采取一些行动。

```sh
npm run start
```

## 代码演示

### Context

第一行是我们要为 Relayer 应用程序提供的 `Context` 的类型声明。

```ts
export type MyRelayerContext = LoggingContext &
  StorageContext &
  SourceTxContext &
  TokenBridgeContext &
  StagingAreaContext &
  WalletContext;
```

这种类型，我们稍后会使用它来参数化通用的 `RelayerApp`，它指定了 `RelayerApp` 可用的 `Context` 对象的联合。

由于 `Context` 对象会被传递给处理器的回调函数，因此提供一个类型参数化的类型定义可以确保在 `Context` 对象的回调中使用相应的字段。

### 应用程序创建

接下来，我们实例化一个 `RelayerApp`，通过 `Context` 类型对其进行参数化。

```ts
const app = new RelayerApp<MyRelayerContext>(Environment.TESTNET);
```

在本示例中，我们定义了一个类 `ApiController`，用于提供我们将传递给处理器回调的方法。请注意，`ctx` 参数类型与我们定义的 `Context` 类型一致。

```ts
export class ApiController {
  processFundsTransfer = async (ctx: MyRelayerContext, next: Next) => {
    // ...
  };
}
```

这不是必须的，但这样的模式有助于随着 `RelayerApp` 的发展来组织代码库。

我们实例化我们的控制器类并通过传递 Spy URL、存储层和日志记录器开始配置我们的应用程序。

```ts
const fundsCtrl = new ApiController();
const namespace = "simple-relayer-example";
const store = new RedisStorage({
  attempts: 3,
  namespace, // used for redis key namespace
  queueName: "relays",
});

// ...

app.spy("localhost:7073");
app.useStorage(store);
app.logger(rootLogger);
```

### Middleware

配置好应用程序后，我们就可以开始添加 `Middleware` 了。

`Middleware` 是我们希望应用于每个收到的 VAA 的功能组件的术语。

`RelayerApp` 定义了一个 `use` 方法，该方法接受一个或多个 `Middleware` 实例，并且也用 `Context` 类型进行参数化。

```ts
use(...middleware: Middleware<ContextT>[] | ErrorMiddleware<ContextT>[])
```

通过向 `use` 方法传递某些 `Middleware` 的实例，我们将其添加到 `RelayerApp` 调用的管道处理程序中。

> 请注意，这里添加 `Middleware` 的顺序很重要，因为 VAA 会以相同的顺序通过每个中间件。

```ts
// we want an instance of a logger available on the context
app.use(logging(rootLogger));
// we want to check for any missed VAAs if we receive out of order sequence ids
app.use(missedVaas(app, { namespace: "simple", logger: rootLogger }));
// we want to apply the chain specific providers to the context passed downstream
app.use(providers());
// enrich the context with details about the token bridge
app.use(tokenBridgeContracts());
// ensure we use redis safely in a concurrent environment
app.use(stagingArea());
// make sure we have the source tx hash
app.use(sourceTx());
```

### 订阅

通过我们的 `Middleware` 设置，我们可以配置一个订阅，用了仅接收我们关心的 VAAs。

在这里，我们设置了一个订阅请求，以接收来自 Solana 的 VAAs，且由以下地址发出 `DZnkkTmCiFWfYTfT41X3Rd1kDgozqzxWaHqsw6W4x2oe`。

在收到与此过滤器匹配的 VAA 后，将调用 `fundsCtrl.processFundsTransfer` 回调函数，并传入一个已经通过我们之前设置的 `Middleware` 处理的 `Context` 对象实例。

```ts
app
  .chain(CHAIN_ID_SOLANA)
  .address(
    "DZnkkTmCiFWfYTfT41X3Rd1kDgozqzxWaHqsw6W4x2oe",
    fundsCtrl.processFundsTransfer,
  );
```

要订阅更多链或地址，可以重复此模式，或者使用一个从 `ChainId` 映射到 `Address` 的对象调用 `multiple` 方法。

```ts
app.multiple(
  {
    [CHAIN_ID_SOLANA]: "DZnkkTmCiFWfYTfT41X3Rd1kDgozqzxWaHqsw6W4x2oe"
    [CHAIN_ID_ETH]: ["0xabc1230000000...","0xdef456000....."]
  },
  fundsCtrl.processFundsTransfer,
);
```

### 错误处理

我们应用的最后一个 `Middleware` 是一个错误处理程序，它将在其前面的任何 `Middleware` 组件抛出错误时被调用。

请注意，该函数有 3 个参数，这提示 `RelayerApp` 应该使用它来处理错误。

```ts
app.use(async (err, ctx, next) => {
  ctx.logger.error("error middleware triggered");
});
```

### 启动监听

最后，调用 `app.listen()` 将启动 `RelayerApp`，发起订阅请求并按我们配置的方式处理 VAAs。

`listen` 方法是异步的，使用 `await` 等待它将阻塞程序，直到程序因不可恢复的错误而结束、进程被终止，或使用 `app.stop()` 停止应用程序。

### Bonus UI

在本例中，我们使用 `koa` 提供了默认 UI。

当程序运行时，你可以打开浏览器 `http://localhost:3000/ui` 来查看运行中程序的详细信息。

## 更进一步

其中包含的默认功能可能无法满足您的使用要求。

如果你想要应用一些特定的中间处理步骤，可以考虑实现一些自定义的 `Middleware`。在为 `RelayerApp` 类型参数化时，确保包含适当的 `Context`，以便将你希望添加的任何字段加入到传递给下游 `Middleware` 的 `Context` 对象中。

如果您希望使用 redis 以外的存储层，只需实现[存储](https://github.com/wormhole-foundation/relayer-engine/blob/main/relayer/storage/storage.ts)接口即可。

{% hint style="info" %}
### Wormhole 集成完成了吗？

请告诉我们，这样我们就可以将你的项目列入我们的生态目录，并将你介绍给我们的全球多链社区！

[立即联系！](https://forms.clickup.com/45049775/f/1aytxf-10244/JKYWRUQ70AUI99F32Q)
{% endhint %}
