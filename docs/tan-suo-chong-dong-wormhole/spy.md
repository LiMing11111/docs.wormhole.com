<!--
 * @Author: “crislee” ‘505267309@qq.com’
 * @Date: 2024-04-18 18:28:03
 * @LastEditors: “crislee” ‘505267309@qq.com’
 * @LastEditTime: 2024-04-19 13:15:21
 * @FilePath: /docs.wormhole.com/docs/tan-suo-chong-dong-wormhole/spy.md
 * @Description: 这是默认设置,请设置`customMade`, 打开koroFileHeader查看配置 进行设置: https://github.com/OBKoro1/koro1FileHeader/wiki/%E9%85%8D%E7%BD%AE
-->
# Spy

### Spy

在 wormhole 上下文中，Spy 是一个守护进程，订阅[守护网络](guardian.md)中的 gossiped 消息。

通过 gossiped 可获取的消息包括：

* [VAAs](vaa.md)
* [Observation](../reference/glossary.md#observation)
* Guardian 心跳

Spy 的源代码可以在 [Github](https://github.com/wormhole-foundation/wormhole/blob/main/node/cmd/spy/spy.go) 上找到。

{% hint style="info" %}
Spy 没有内置持久性层，所以通常与像 Redis 或 SQL 数据库之类的东西配对使用以记录相关消息。
{% endhint %}

### Use

要在本地启动一个 Spy ，运行以下的 docker 命令。

{% tabs %}
{% tab title="Testnet" %}
```sh
docker run --platform=linux/amd64 \
    -p 7073:7073 \
    --entrypoint /guardiand ghcr.io/wormhole-foundation/guardiand:latest \
    spy \
    --nodeKey /node.key \
    --spyRPC "[::]:7073" \
    --env testnet
```
可选地操作，添加 flags 以跳过任何具有无效签名的 VAAs

```sh
--ethRPC https://sepolia.drpc.org/
--ethContract 0x4a8bc80Ed5a4067f1CCf107057b8270E0cC11A78
```
{% endtab %}

{% tab title="Mainnet" %}
```sh
docker run --platform=linux/amd64 \
    -p 7073:7073 \
    --entrypoint /guardiand ghcr.io/wormhole-foundation/guardiand:latest \
    spy \
    --nodeKey /node.key \
    --spyRPC "[::]:7073" \
    --env mainnet
```

可选地操作，添加 flags 以跳过任何具有无效签名的 VAAs

```sh
--ethRPC https://eth.drpc.org
--ethContract 0x98f3c9e6E3fAce36bAAd05FE09d375Ef1464288B
```
{% endtab %}
{% endtabs %}

一旦运行起来一个[ gRPC ](https://grpc.io/)客户端（即你的程序）即可以订阅过滤后的消息流。

为了生成 gRPC 服务的客户端，请使用[此规范的 Spy 文件](https://github.com/wormhole-foundation/wormhole/blob/main/proto/spy/v1/spy.proto)。

{% hint style="info" %}
如果使用 JavaScript/TypeScript，[Spydk](https://www.npmjs.com/package/@certusone/wormhole-spydk) 可以使设置客户端更容易。
{% endhint %}

### 另请参阅

[中继器引擎](https://github.com/wormhole-foundation/relayer-engine)实现了一个客户端和持久层，用于接收来自 Spy 订阅的消息。