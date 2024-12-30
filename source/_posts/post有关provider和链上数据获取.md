---
title: 有关provider和链上数据获取
date: 2024-12-30 17:27:31
tags: provider on-chain
---



## 简介

**provider** 是一个用于与区块链网络交互的对象，主要负责提供区块链的数据和与智能合约的通信。它是连接你本地应用（如前端、脚本）与区块链节点之间的桥梁。


## 实现

拥有自己的节点，这可能需要大量的时间来配置，所以我们会去找第三方的 node provider 


![](../attachment/Pasted%20image%2020241225173327.png)


 这是一个 DApp 架构中特殊的角色，它负责与区块链进行通信，并进行合约的读/写操作。他是钱包和区块链的中枢。

可以分为 2 种
- InjectProvider（web 3 Provider） 主流 metamask
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

另一种写法

```js
const { ethers } = require("ethers");

// 配置以太坊网络的 RPC URL，例如 Infura 或 Alchemy 的服务地址
const provider = new ethers.providers.JsonRpcProvider("https://mainnet.infura.io/v3/YOUR_INFURA_PROJECT_ID");
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


## 获取以太坊数据

### 前置条件

- 了解 url，uri，restful 架构，json-rpc 架构。
- 了解 web service

#### Web service 

A web service is a way to share data with two seperate systems. It can be the same called web api.
It open contains many protocal like 
应用层协议	HTTP/HTTPS, WebSocket, gRPC, CoAP，HTTP/2	主要用于客户端和服务器之间的通信。
数据格式协议	JSON-RPC, SOAP, Protocol Buffers	定义数据的序列化格式和消息传递方式。
传输协议	TCP, UDP, QUIC	提供底层数据传输支持。
特定场景协议	GraphQL, MQTT, OData, SSE	为特定需求（如实时通信、数据查询）设计的协议。

我们可以着重看一下应用层 HTTP/HTTPS, 数据格式 JSON-RPC。


#### 资源获取

Web 中资源是很重要的概念，我们需要能够定位互联网上的资源，一种是 uri，一种是 url。

*Url* 是基于 http 协议的。比如 `https://example.com/api/data`
*Uri* 则是 url 的父级，除了 url 他还包括 
文本：`file:///C:/Users/Username/Documents/file.txt`
图片数据, 将数据内嵌进去: `data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUA...`
相对 url： 相对路径。

#### 方法调用

Reference:https://stackoverflow.com/questions/50394931/what-is-difference-between-json-rpc-and-json-api

除了资源的获取，我们通常还需要调用服务端写好的方法。这个时候我们一般有几种调用方式

 *json-rpc 数据结构*，使用 json 来调用获取结果，他是用来方法调用返回 json 的，但如果我们想获取资源，我们可以在 result 中放资源的 uri 或者资源本身，只是不建议这么做。
他只是远程过程调用规范，他与传输无关，你可以像非常常见的那样在 HTTP 上运行它，你也可以使用它在套接字上运行，或者使用任何其他你认为合适的传输方式

```json

{
  "jsonrpc": "2.0",
  "method": "getPageHtml",
  "params": {"page": "home"},
  "id": 1
}
返回
{
  "jsonrpc": "2.0",
  "result": "<html><head><title>Home Page</title></head><body><h1>Welcome to the Home Page</h1></body></html>",
  "id": 1
}
```


*Restful 风格 api* ，与 Json-Rpc 不同，它需要你在 HTTP 服务器上托管。你不能通过它在客户端调用函数。你不能使用非 HTTP 传输协议运行它。作为基于 REST 的规范，它在提供资源信息方面表现出色。如果你想使用一个基于创建、读取、更新、删除某些资源集合的 API，那么这可能是一个不错的选择。
也就是他可以通过方法来获取资源
并且 REST 使用资源路径（URL）来标识资源，例如：/users/123 可以直观地表示某个用户的资源。
这种设计方式使得 REST 的 API 更易于理解和直观。例如，通过浏览器直接访问 https://example.com/users/123，可以轻松获取资源内容

他俩平常都是在 http 协议上运行的，通过 http 的方式来获取数据。


### Service 模块

很多节点比如 infura，alchemy 都可以用 web api 来让 Dapp ,wallet 获取区块链的数据。这里我将介绍 infura 节点。

![](../attachment/Pasted%20image%2020241230145917.png)

Infura 提供 json-rpc 和 restful 两种方式来访问网络数据。并且底下列出了支持的网络。不是所有网络都被节点兼容。左边是接口的说明和支持的网络。这些接口两种风格。

Restful ：``https://gas.api.infura.io/v3/${process.env.INFURA_API_KEY}/networks/${chainId}/suggestedGasFees``

Json-rpc: `https://mainnet.infura.io.infura.io/v3/<YOUR-API-KEY>`

拿 ethereum 的数据获取举例，我们可以通过 axios，fetch 来获取，通过输入 url，传递的 json 实现。

![](../attachment/Pasted%20image%2020241230150117.png)

同时可以用 ether. Js 第三方库，用他们封装好的 provider 来做数据的获取。

![](../attachment/Pasted%20image%2020241230150408.png)


由于我们是获取数据，我们可以调用 `eth_call` 的方法，用于调用智能合约的“只读”函数（即不更改区块链状态的函数）。它允许你查询智能合约的状态或读取数据，但不会消耗 gas 费用，因为它不会触发区块链上的交易。


### 使用 json-rpc 和链上合约交互

```js
const baseUrl = 'https://mainnet.infura.io/v3/1fbfb41663cb4efe83ddcd36082586d0';
const data = {
  jsonrpc: '2.0',
  method: 'eth_call',
  params: [
    {
      to: collection,
      data: "0x8da5cb5b"
    }
  ],
  id: 1
};

const res = await fetch(baseUrl, {
  method: "POST",
  headers: {
    'Content-Type': "application/json",
  },
  body: JSON.stringify(data),
});

if (!res.ok) {
  throw new Error('Failed to fetch NFTs');
}

const jsonData = await res.json();
```

这是最原始的方法，通过直接用 eth. Call 方法，调用智能合约的方法。

*我们可以通过 json-rpc 和以太坊的节点交互，但是和合约交互，我们必须有合约的 abi，并写在 json 里*
Data 就是智能合约方法签名和编码参数的哈希值。参见以太坊合约 ABI 规范。[Ethereum contract ABI specification](https://docs.soliditylang.org/en/latest/abi-spec.html).

我这里写的是一个 owner () 的方法，可以在 etherscan 看到想要测试的合约方法并且获得方法的 hash。

#### 第三方库

当然我们也可以使用封装好的 ether, webjs 等第三方库。他们的使用方法有所不同。

```js
const fetch = require("node-fetch")
const { Web3 } = require("web3")

const web3 = new Web3(
  new Web3.providers.HttpProvider("https://mainnet.infura.io/v3/<YOUR-API-KEY>")
)

const tokenURIABI = [
  {
    inputs: [
      {
        internalType: "uint256",
        name: "tokenId",
        type: "uint256",
      },
    ],
    name: "tokenURI",
    outputs: [
      {
        internalType: "string",
        name: "",
        type: "string",
      },
    ],
    stateMutability: "view",
    type: "function",
  },
]

const tokenContract = "0xbc4ca0eda7647a8ab7c2061c2e118a18a936f13d" // BAYC contract address
const tokenId = 101 // A token we'd like to retrieve its metadata of

const contract = new web3.eth.Contract(tokenURIABI, tokenContract)

async function getNFTMetadata() {
  const result = await contract.methods.tokenURI(tokenId).call()

  console.log(result) // ipfs://QmeSjSinHpPnmXmspMjwiXyN6zS4E9zccariGR3jxcaWtq/101

  const ipfsURL = addIPFSProxy(result)

  const response = await fetch(ipfsURL)
  const metadata = await response.json()
  console.log(metadata) // Metadata in JSON

  const image = addIPFSProxy(metadata.image)
}

getNFTMetadata()

function addIPFSProxy(ipfsHash) {
  const URL = "https://<YOUR_SUBDOMAIN>.infura-ipfs.io/ipfs/"
  const hash = ipfsHash.replace(/^ipfs?:\/\//, "")
  const ipfsURL = URL + hash

  console.log(ipfsURL) // https://<subdomain>.infura-ipfs.io/ipfs/<ipfsHash>
  return ipfsURL
}
```


#### 使用封装好的 api

我们目前为止用的是原生的 json-rpc api。但我们也可以用集成包装好的 api 完成我们更复杂的内容，我觉得这一方面 alchemy 做的更好，他们提供了更丰富的 restful api 帮我们实现更加复杂的内容。并且他们的开发文档更加详细 

可以参考 [infura or alchemy](https://www.reddit.com/r/ethdev/comments/10dnpxf/infura_or_alchemy_which_is_better/)
