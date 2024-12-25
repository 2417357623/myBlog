---
title: buy-me-coffee-dapp
date: 2024-12-25 21:41:14
tags:
---

## Reference

[buy-me-a-coffee](https://www.youtube.com/watch?v=cxxKdJk55Lk&t=1016s)


## 技术栈

Solidity，hardhat, react, ether


## 编写合约

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.28;

contract BuyMeACoffee {
    //Event to emit when a memo is created
    event NewMemo(
        address indexed from,
        uint256 timestamp,
        string name,
        string message
    );

    struct Memo{
        address from;
        uint256 timestamp;
        string name;
        string message;
    }

    Memo[] memos;

    address payable public owner;

    constructor(){
        owner = payable(msg.sender);
    }

    function buyCoffee(string memory _name,string memory _message) payable public {
        require(msg.value > 0,"can't buy coffee with 0 eth");

        memos.push(Memo(
            msg.sender,
            block.timestamp,
            _name,
            _message
        ));

        emit NewMemo(
            msg.sender,
            block.timestamp,
            _name,
            _message
        );
    }

    function withdrawTips() public {
        require(owner.send(address(this).balance));

        
    }

    function getMemos() public view returns(Memo[] memory){
        return memos;
    }
}
```

涉及了资金的转入，转出，事件的发送


## 部署验证测试合约

### 本地测试的代码

```js
const hre = require("hardhat");

//returns the Ethereum balance of a given address.
async function getBalance(address){
    //hardhat-network provider. get balance from this network
    const balanceBigInt = await hre.ethers.provider.getBalance(address);
    return hre.ethers.formatEther(balanceBigInt);
}

async function printBalance(addresses) {
    let idx = 0;
    for(const address of addresses){
        console.log(`address ${idx} balance:`,await getBalance(address));
        idx++
    }
}

async function printMemos(memos) {
   for(const memo of memos){
    console.log(`at ${memo.timestamp} ${memo.name} ,${memo.from} said ${memo.message}`)
   } 
}


async function main(){
    //get example account 
    const [owner,tipper,tipper2,tipper3] = await hre.ethers.getSigners();

    console.log(`owner is :`,owner.address);

    //get the contract to deploy
    const BuyMeACoffee = await hre.ethers.getContractFactory("BuyMeACoffee");
    const buyMeACoffee = await BuyMeACoffee.deploy();
    await buyMeACoffee.waitForDeployment();
    console.log(
        `contract has been deployed successfully, contract address is ${buyMeACoffee.target}`
    );

    console.log(`owner is :`,await buyMeACoffee.owner());

    //check the balance before the coffee purchase
    const addresses = [owner.address,tipper.address,buyMeACoffee.target]
    console.log("== start ==");
    await printBalance(addresses);
    
    //by owner a coffee;
    const tip = {value:hre.ethers.parseEther("1")};

    //connect wallet to contract. call the contract function
    await buyMeACoffee.connect(tipper).buyCoffee("ly","hello",tip);
    await buyMeACoffee.connect(tipper2).buyCoffee("ly2","hello2",tip);
    await buyMeACoffee.connect(tipper3).buyCoffee("ly3","hello3",tip);

    console.log("== bought coffee ==");
    await printBalance(addresses);

    //withdraw funds
    await buyMeACoffee.connect(owner).withdrawTips();
    console.log("== withdraw tips ==");
    await printBalance(addresses);

    //read all memos left for the owner
    console.log("== memos ==");
    const memos = await buyMeACoffee.getMemos();
    printMemos(memos)
    
}

main()
  .then()
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });

```

使用了本地测试网络提供的账户，本地提供的 provider 用来获取本地测试网的数据。并且我们是部署到了本地的测试网。所以我们无需做任何配置，一切都是默认的。

### 以太坊测试网的代码

我们仍然需要将代码部署到以太坊测试网，这样所有人可以测试他，并且用假的以太币，并且和别的合约进行交互。

钱包：metamask 代替默认的假钱包
Provider：alchemy 代替默认的假 provider，来进行对链的写入读取数据，和钱包的交互，执行合约代码。。。这样我们不需要维护任何的节点。

```js
const hre = require("hardhat");

async function main(){
        //get the contract to deploy
        const BuyMeACoffee = await hre.ethers.getContractFactory("BuyMeACoffee");
        const buyMeACoffee = await BuyMeACoffee.deploy();
        await buyMeACoffee.waitForDeployment();
        console.log(
            `contract has been deployed successfully, contract address is ${buyMeACoffee.target}`
        );
    
        console.log(`owner is :`,await buyMeACoffee.owner());
}

main()
  .then()
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

这里我们需要对 hardhat. Config 进行配置

```js
require("@nomicfoundation/hardhat-toolbox");
require('dotenv').config() //为了可以使用.env里定义的变量

const SEPOLIA_URL = process.env.SEPOLIA_URL;
const PRIVATE_KEY = process.env.PRIVATE_KEY;
const API_KEY = process.env.API_KEY;

/** @type import('hardhat/config').HardhatUserConfig */
module.exports = {
  solidity: "0.8.28",
  networks:{
    sepolia:{
      url:SEPOLIA_URL,
      accounts:[PRIVATE_KEY],
      chainId:11155111
    }
  },
  etherscan: {
    // Your API key for Etherscan,，允许你在 Hardhat 中自动验证和上传智能合约源代码到 Etherscan。
    //使用 Etherscan 的验证功能可以提高合约的透明度，允许其他开发者和用户查看和审核你的合约代码，增加可信度。
    apiKey: {
      sepolia:API_KEY
    }
  },
};
```

至此，合约被部署到了 sepolia 测试网，那前端如何设计页面并和合约进行互动呢。


## 客户端开发

### 浏览器环境的合约交互

我们使用 repl 快速搭建环境和前端项目。利用 React，jsx 来搭建我们的客户端。使钱包用户可以方便的和合约进行交互。

```jsx
import abi from '../utils/BuyMeACoffee.json';
import { ethers } from "ethers";
import Head from 'next/head'
import Image from 'next/image'
import React, { useEffect, useState } from "react";
import styles from '../styles/Home.module.css'

export default function Home() {
  // Contract Address & ABI
  const contractAddress = "0xd234976128226C683A3dcc9De834BFC5E2a49586";
  const contractABI = abi.abi;

  // Component state
  const [currentAccount, setCurrentAccount] = useState("");
  const [name, setName] = useState("");
  const [message, setMessage] = useState("");
  const [memos, setMemos] = useState([]);

  const onNameChange = (event) => {
    setName(event.target.value);
  }

  const onMessageChange = (event) => {
    setMessage(event.target.value);
  }

  // Wallet connection logic
  const isWalletConnected = async () => {
    try {
      const { ethereum } = window;

      const accounts = await ethereum.request({method: 'eth_accounts'})
      console.log("accounts: ", accounts);

      if (accounts.length > 0) {
        const account = accounts[0];
        console.log("wallet is connected! " + account);
      } else {
        console.log("make sure MetaMask is connected");
      }
    } catch (error) {
      console.log("error: ", error);
    }
  }

  const connectWallet = async () => {
    try {
      const {ethereum} = window;

      if (!ethereum) {
        console.log("please install MetaMask");
      }

      const accounts = await ethereum.request({
        method: 'eth_requestAccounts'
      });

      setCurrentAccount(accounts[0]);
    } catch (error) {
      console.log(error);
    }
  }

  const buyCoffee = async () => {
    try {
      const {ethereum} = window;

      if (ethereum) {
        const provider = new ethers.providers.Web3Provider(ethereum, "any");
        console.log(provider);
        const signer = provider.getSigner();
        console.log(signer)
        const buyMeACoffee = new ethers.Contract(
          contractAddress,
          contractABI,
          signer
        );

        console.log("buying coffee..")
        const coffeeTxn = await buyMeACoffee.buyCoffee(
          name ? name : "anon",
          message ? message : "Enjoy your coffee!",
          {value: ethers.utils.parseEther("0.001")}
        );

        await coffeeTxn.wait();

        console.log("mined ", coffeeTxn.hash);

        console.log("coffee purchased!");

        // Clear the form fields.
        setName("");
        setMessage("");
      }
    } catch (error) {
      console.log(error);
    }
  };

  // Function to fetch all memos stored on-chain.
  const getMemos = async () => {
    try {
      const { ethereum } = window;
      if (ethereum) {
        const provider = new ethers.providers.Web3Provider(ethereum);
        const signer = provider.getSigner();
        const buyMeACoffee = new ethers.Contract(
          contractAddress,
          contractABI,
          signer
        );
        
        console.log("fetching memos from the blockchain..");
        const memos = await buyMeACoffee.getMemos();
        console.log("fetched!");
        setMemos(memos);
      } else {
        console.log("Metamask is not connected");
      }
      
    } catch (error) {
      console.log(error);
    }
  };
  
  useEffect(() => {
    let buyMeACoffee;
    isWalletConnected();
    getMemos();

    // Create an event handler function for when someone sends
    // us a new memo.
    const onNewMemo = (from, timestamp, name, message) => {
      console.log("Memo received: ", from, timestamp, name, message);
      setMemos((prevState) => [
        ...prevState,
        {
          address: from,
          timestamp: new Date(timestamp * 1000),
          message,
          name
        }
      ]);
    };

    const {ethereum} = window;

    // Listen for new memo events.
    if (ethereum) {
      const provider = new ethers.providers.Web3Provider(ethereum, "any");
      const signer = provider.getSigner();
      buyMeACoffee = new ethers.Contract(
        contractAddress,
        contractABI,
        signer
      );

      buyMeACoffee.on("NewMemo", onNewMemo);
    }

    return () => {
      if (buyMeACoffee) {
        buyMeACoffee.off("NewMemo", onNewMemo);
      }
    }
  }, []);
  
  return (
    <div className={styles.container}>
      <Head>
        <title>Buy Albert a Coffee!</title>
        <meta name="description" content="Tipping site" />
        <link rel="icon" href="/favicon.ico" />
      </Head>

      <main className={styles.main}>
        <h1 className={styles.title}>
          Buy Albert a Coffee!
        </h1>
        
        {currentAccount ? (
          <div>
            <form>
              <div>
                <label>
                  Name
                </label>
                <br/>
                
                <input
                  id="name"
                  type="text"
                  placeholder="anon"
                  onChange={onNameChange}
                  />
              </div>
              <br/>
              <div>
                <label>
                  Send Albert a message
                </label>
                <br/>

                <textarea
                  rows={3}
                  placeholder="Enjoy your coffee!"
                  id="message"
                  onChange={onMessageChange}
                  required
                >
                </textarea>
              </div>
              <div>
                <button
                  type="button"
                  onClick={buyCoffee}
                >
                  Send 1 Coffee for 0.001ETH
                </button>
              </div>
            </form>
          </div>
        ) : (
          <button onClick={connectWallet}> Connect your wallet </button>
        )}
      </main>

      {currentAccount && (<h1>Memos received</h1>)}

      {currentAccount && (memos.map((memo, idx) => {
        return (
          <div key={idx} style={{border:"2px solid", "borderRadius":"5px", padding: "5px", margin: "5px"}}>
            <p style={{"fontWeight":"bold"}}>"{memo.message}"</p>
            <p>From: {memo.name} at {memo.timestamp.toString()}</p>
          </div>
        )
      }))}

      <footer className={styles.footer}>
        <a
          href="https://alchemy.com/?a=roadtoweb3weektwo"
          target="_blank"
          rel="noopener noreferrer"
        >
          Created by @thatguyintech for Alchemy's Road to Web3 lesson two!
        </a>
      </footer>
    </div>
  )
}

```

首先注意这是浏览器环境，所以获取 provider 和刚才的 node 有所不同。

其次这里涉及到了链接钱包，获取钱包的签名者，获取钱包的 provider，合约实例生成（3 个参数），合约函数调用。有些和 node 那边不一样，有些是一样的。

### 我再提供一个 node 环境的脚本文件，主要用来回收资金

```js
// scripts/withdraw.js

const hre = require("hardhat");
const abi = require("../artifacts/contracts/BuyMeACoffee.sol/BuyMeACoffee.json");

async function getBalance(provider, address) {
  const balanceBigInt = await provider.getBalance(address);
  return hre.ethers.utils.formatEther(balanceBigInt);
}

async function main() {
  // Get the contract that has been deployed to Goerli.
  const contractAddress="0xDBa03676a2fBb6711CB652beF5B7416A53c1421D";
  const contractABI = abi.abi;

  // Get the node connection and wallet connection.
  const provider = new hre.ethers.providers.JsonRpcProvider("goerli", process.env.GOERLI_API_KEY);

  // Ensure that signer is the SAME address as the original contract deployer,
  // or else this script will fail with an error.
  const signer = new hre.ethers.Wallet(process.env.PRIVATE_KEY, provider);

  // Instantiate connected contract.
  const buyMeACoffee = new hre.ethers.Contract(contractAddress, contractABI, signer);

  // Check starting balances.
  console.log("current balance of owner: ", await getBalance(provider, signer.address), "ETH");
  const contractBalance = await getBalance(provider, buyMeACoffee.address);
  console.log("current balance of contract: ", await getBalance(provider, buyMeACoffee.address), "ETH");

  // Withdraw funds if there are funds to withdraw.
  if (contractBalance !== "0.0") {
    console.log("withdrawing funds..")
    const withdrawTxn = await buyMeACoffee.withdrawTips();
    await withdrawTxn.wait();
  } else {
    console.log("no funds to withdraw!");
  }

  // Check ending balance.
  console.log("current balance of owner: ", await getBalance(provider, signer.address), "ETH");
}

// We recommend this pattern to be able to use async/await everywhere
// and properly handle errors.
main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```


### 比较（重要）

| 特性              | 第一段代码（`withdraw.js`）   | 第二段代码（`Web3Provider`） |      |
| --------------- | ---------------------- | --------------------- | ---- |
| **运行环境**        | Node.js + Hardhat      | 浏览器 + MetaMask        |      |
| **Provider 类型** | `JsonRpcProvider`      | `Web3Provider`        |      |
| **签名者来源**       | 环境变量中的私钥               | 用户通过 MetaMask 签署      |      |
| **适用场景**        | 后端脚本，自动化操作（如提取资金、批量交易） | 前端交互，用户发起交易（如支付、操作合约） |      |
| **与区块链交互**      | 独立连接节点，通过私钥签名          | 依赖浏览器钱包，通过用户确认并签名     | **** |
