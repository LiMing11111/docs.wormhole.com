# Wormhole Connect: 让桥接变得更容易

### 概述

Wormhole Connect 是一个 React 小工具/部件，可让开发人员直接在 Web 应用程序中轻松、自定义地访问 Wormhole 支持的跨链桥。Connect 支持多种桥接形式，包括原生资产桥接、Portal 包装资产桥接、CCTP USDC 桥接等。Connect 为每种桥接提供了 gas droppoff（交易会为用户留下额外的原生代币，以便他们为后续的链上交互支付 gas）和 gasless 交易（Connect 中继器代表用户支付 gas）。

查看 [Github 存储库](https://github.com/wormhole-foundation/wormhole-connect)！

![Wormhole Connect Screenshot](https://camo.githubusercontent.com/fda29f71df76f388a4e579624e538c876f89c396d2dd6d9486657aa8f9a3a19c/68747470733a2f2f692e696d6775722e636f6d2f735a4a4b7738652e706e67)

{% hint style="success" %}
[Wormhole Typescript SDK](https://docs.wormhole.com/wormhole/reference/sdk-docs) 允许您在您自己的 UI 中实现与 Connect 小部件相同的功能。有关使用 SDK 的更多信息，请查看[文档](https://docs.wormhole.com/wormhole/reference/sdk-docs)。
{% endhint %}

### Demo

Wormhole Connect 已实时部署在多个生产环境下的应用程序中。以下是一些具体案例：

* [Portal Bridge](https://portalbridge.com/)
* [Jupiter](https://jup.ag/bridge/wormhole)
* [Pancake Swap](https://bridge.pancakeswap.finance/wormhole)

### 立即开始

将 Wormhole Connect 添加到现有的 React 应用程序非常容易。

首先，安装 npm 包。

[![npm version](https://img.shields.io/npm/v/@wormhole-foundation/wormhole-connect.svg)](https://www.npmjs.com/package/@wormhole-foundation/wormhole-connect)

```bash
npm i @wormhole-foundation/wormhole-connect
```

现在您可以使用 React 组件：

```javascript
import WormholeConnect from '@wormhole-foundation/wormhole-connect';

function App() {
  return (
    <WormholeConnect />
  );
}
```

#### **替代方案：通过 CDN 托管版本（适用于任何网站）**

如果您不使用 React，您仍然可以使用托管版本将 Connect 嵌入到您的网站上。只需将以下代码复制并粘贴到您的 HTML 正文中：

```html
<!-- Mounting point. Include in <body> -->
<div id="wormhole-connect"></div>

<!-- Dependencies -->
<script type="module" src="https://www.unpkg.com/@wormhole-foundation/wormhole-connect@0.3.0-beta.9-development/dist/main.js" defer></script>
<link rel="https://www.unpkg.com/@wormhole-foundation/wormhole-connect@0.3.0-beta.9-development/dist/main.css" />
```

### 配置

在 [Connect 文档](https://docs.wormhole.com/wormhole/wormhole-connect/overview#configuration)中了解有关配置 Connect 的信息以及更多内容！
