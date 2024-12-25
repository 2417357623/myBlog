---
title: web3重要概念-provider的介绍
date: 2024-12-25 21:49:37
tags:
---



![](attachment/Pasted%20image%2020241225173327.png)


 这是一个 DApp 架构中特殊的角色，它负责与区块链进行通信，并进行合约的读/写操作。他是钱包和区块链的中枢。

可以分为 2 种
- InjectProvider（web3 Provider） 主流 metamask
- JSON-RPC Provider  主流 Infura achemy



*本地网络的 provider*

```js
const provider = ethers.provider; // 默认为 Hardhat 网络的provider
```

*以太坊网络的 provider*，需要提前在 hardhat 配置里面配置好 rpc 的 api-key，我们通过这个 http 地址就可以实现与远程节点的交互。

```js
// 获取 provider 对象，连接到以太坊网络
const provider = ethers.provider; // 默认为 Hardhat 网络，如果使用测试网或主网，需要提供相应的 RPC URL

// 查询地址的余额，返回的是 BigNumber 类型
const balance = await provider.getBalance(address);
```

*Metamask 的 provider*

Metamask uses Infura in the background to connect to the network. So Metamask is a user interface on top of Infura service.
我们只需要在 metamask 中选择默认支持的网（infura 默认支持）或者自主设置 rpc 的自定义网，就可以通过浏览器中已经安装的 `ethereum` 对象，获取当前的网的 provider，这就不需要我们向上面一样，去 config 里面配置 rpc-url。（相当于使用 metamask 提前设置好，我们提前自定义好的 rpc）

这里提供一个在浏览器环境中的 provider 代码

```js
const {ethereum} = window;

const provider = new ethers.providers.Web3Provider(ethereum, "any");
```

>因为 metamask 是浏览器插件，不支持 node。，如果要在 node 环境中使用 metamask，还是像上面两个方式一样，记录私钥和 rpc-url。

Ethereum: 这是一个由浏览器环境（如 MetaMask）提供的对象，它代表当前浏览器中与区块链连接的 Web 3 提供者。你通常在浏览器中通过 window. Ethereum 获取这个对象（MetaMask 会注入这个对象）。
例如，window. Ethereum 是 MetaMask 提供的一个对象，允许网页与 MetaMask 交互，执行签名、发送交易等操作。
"any": 这是 Web 3 Provider 的第二个参数，指定你想要连接的网络。这里 "any" 表示自动选择当前 ethereum 对象所连接的任何网络（如主网、测试网等）。如果没有特别指定，它会选择 ethereum 对象当前连接的网络。
这也就是为什么在选择发送交易前，钱包需要切换到对应的网络上来。