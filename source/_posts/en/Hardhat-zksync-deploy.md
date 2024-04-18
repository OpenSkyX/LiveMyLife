---
title: Zksync Era 使用Hardhat部署只能合约
catalog: true
date: 2022-08-21 14:39:17
subtitle: 最近想在Zksync上部署一个erc20代币，发现直接拿原来的合约没办法直接用remix部署。研究了一下发现目前Zksync Era Mainnet的部署流程不同，需要用到hardhat来部署。
header-img: /img/header_img/lml_bg.jpg
tags:
- ZkSync Era
categories:
- ZkSync Era
---

### 参考资料

> [Zksync Era Mainnet官方文档的Quickstart](https://era.zksync.io/docs/dev/building-on-zksync/hello-world.html#initializing-the-project-deploying-a-smart-contract)
> 
> [Hardhat官方文档](https://hardhat.org/hardhat-runner/docs/getting-started#overview)


### 环境准备

安装yarn或者npm包管理器，最好用 [yarn](https://yarnpkg.com/getting-started/install)

zksync链的钱包里要有足够的eth，可以用官方跨链桥跨过去，钱包私钥记好一会代码里会用

初始化环境和依赖，下面的步骤是新建文件夹和安装对应的依赖

```shell
mkdir greeter-example
cd greeter-example

# For Yarn
yarn init -y
yarn add -D typescript ts-node ethers@^5.7.2 zksync-web3 hardhat @matterlabs/hardhat-zksync-solc @matterlabs/hardhat-zksync-deploy
```

### 部署流程


新建配置文件，命名为"hardhat.config.ts"。官网的文档只有测试网的配置，我在这里加上了我们要用的主网配置，默认在主网配置

```typescript
import "@matterlabs/hardhat-zksync-deploy";
import "@matterlabs/hardhat-zksync-solc";

module.exports = {
    zksolc: {
        version: "1.3.5",
        compilerSource: "binary",
        settings: {},
    },
    defaultNetwork: "zkMainnet",

    networks: {
        zkSyncTestnet: {
            url: "https://zksync2-testnet.zksync.dev",
            ethNetwork: "goerli",
            zksync: true,
        },
        zkMainnet: {
            url: "https://zksync2-mainnet.zksync.io",
            ethNetwork: "mainnet",
            zksync: true,
        }
    },
    solidity: {
        version: "0.8.17",
    },
};
```

建两个文件夹 一个叫contracts，一个叫deploy

在contracts文件夹下建立Greeter.sol文件，代码如下（如果是部署erc20合约就贴你自己的erc20合约）：

```solidity
//SPDX-License-Identifier: Unlicense
pragma solidity ^0.8.17;

contract Greeter {
    string private greeting;

    constructor(string memory _greeting) {
        greeting = _greeting;
    }

    function greet() public view returns (string memory) {
        return greeting;
    }

    function setGreeting(string memory _greeting) public {
        greeting = _greeting;
    }
}
```

编译。这一步在国内容易踩坑，原因是solc的代理问题，这里代理上需要用http代理，如果用https代理是下不了的

如果是在国外直接就下面这样编译：

```shell
yarn hardhat compile
```

国内需要挂代理要这样：

```text
http_proxy=http://127.0.0.1:7890 yarn hardhat compile

```

这个代理的代码根据自己的代理设置来，7890是我自己的端口，实际上要替换成你自己挂的梯子的本地端口

5.在deploy文件夹下创建deploy.ts文件，下面是部署脚本。这里代码我删掉了领取测试网gas的代码，因为主网我们已经跨链过来资金了，主网是领不了gas的，否则会报错。

记得将代码里的替换为自己的私钥

```typescript
import { Wallet, utils } from "zksync-web3";
import * as ethers from "ethers";
import { HardhatRuntimeEnvironment } from "hardhat/types";
import { Deployer } from "@matterlabs/hardhat-zksync-deploy";

// An example of a deploy script that will deploy and call a simple contract.
export default async function (hre: HardhatRuntimeEnvironment) {
  console.log(`Running deploy script for the Greeter contract`);

  // Initialize the wallet.
  const wallet = new Wallet("<WALLET-PRIVATE-KEY>");

  // Create deployer object and load the artifact of the contract you want to deploy.
  const deployer = new Deployer(hre, wallet);
  const artifact = await deployer.loadArtifact("Greeter");

  // Estimate contract deployment fee
  const greeting = "Hi there!";
  const deploymentFee = await deployer.estimateDeployFee(artifact, [greeting]);


  // Deploy this contract. The returned object will be of a `Contract` type, similarly to ones in `ethers`.
  // `greeting` is an argument for contract constructor.
  const parsedFee = ethers.utils.formatEther(deploymentFee.toString());
  console.log(`The deployment is estimated to cost ${parsedFee} ETH`);

  const greeterContract = await deployer.deploy(artifact, [greeting]);

  //obtain the Constructor Arguments
  console.log("constructor args:" + greeterContract.interface.encodeDeploy([greeting]));

  // Show the contract info.
  const contractAddress = greeterContract.address;
  console.log(`${artifact.contractName} was deployed to ${contractAddress}`);
}
```

### 部署代码

```shell
yarn hardhat deploy-zksync

```