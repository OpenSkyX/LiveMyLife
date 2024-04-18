---
title: Smart-Contract | How to use Hardhat to deploy contracts to zkSync L2
catalog: true
date: 2023-07-31 12:34:17
subtitle: 智能合约 | 如何使用Hardhat部署合约到zkSync L2
header-img: /img/header_img/lml_bg.jpg
tags:
- Web3-Develop
categories:
- Web3-Develop
---

## 快速开始

#### 本次教程将向你展示如何将智能合约部署到zkSync，并使用ZkSync开发工具构建与其交互的dApp.

#### 这就是我们要做的：

- 在ZkSync Era测试网上构建、部署和验证存储问候消息的智能合约。
  
- 构建一个检索和更新问候消息的 dApp。
  
- 允许用户通过应用程序更改智能合约上的问候语。
  
- 向您展示如何实现测试网 Paymaster，允许用户使用 ERC20 代币而不是 ETH 支付交易费用。

## 先决条件

- L1 上有足够 Göerli 的钱包，ETH用于支付 zkSync 的过渡资金和部署智能合约。
  
- 测试网付款人需要 zkSync 上的 ERC20 代币。我们建议使用zkSync 门户中的水龙头。

## 构建并部署Greeter合约

### 初始化项目

#### 1：安装zkSync CLI：

```shell
npm install -g zksync-cli@latest
```

#### 2: 通过运行以下命令搭建一个新项目：

```shell
zksync-cli create greeter-example
```

###### 这将创建一个新的 zkSync Era 项目，该项目greeter-example包含基本Greeter合约以及所有 zkSync 插件和配置。

#### 3: 导航到项目目录：

```shell
cd greeter-example
```

#### 4:要配置您的私钥，请复制.env.example文件，将副本重命名为.env，然后添加您的钱包私钥。

```text
WALLET_PRIVATE_KEY=0xabcdef12345....
```

### 编译并部署 Greeter 合约

#### 我们将所有智能合约的*.sol文件存储在该contracts文件夹中。该deploy文件夹包含与部署相关的所有脚本。

#### 1:包含的contracts/Greeter.sol合同有以下代码：

```solidity
//SPDX-License-Identifier: Unlicense
pragma solidity ^0.8.8;

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

#### 2:使用以下命令编译合约：

```shell
yarn hardhat compile
```

#### 3：zkSync -CLI还提供了一个部署脚本/deploy/deploy-greeter.ts

```typescript
import { Wallet, utils } from "zksync-web3";
import * as ethers from "ethers";
import { HardhatRuntimeEnvironment } from "hardhat/types";
import { Deployer } from "@matterlabs/hardhat-zksync-deploy";

// load env file
import dotenv from "dotenv";
dotenv.config();

// load wallet private key from env file
const PRIVATE_KEY = process.env.WALLET_PRIVATE_KEY || "";

if (!PRIVATE_KEY) throw "⛔️ Private key not detected! Add it to the .env file!";

// An example of a deploy script that will deploy and call a simple contract.
export default async function (hre: HardhatRuntimeEnvironment) {
  console.log(`Running deploy script for the Greeter contract`);

  // Initialize the wallet.
  const wallet = new Wallet(PRIVATE_KEY);

  // Create deployer object and load the artifact of the contract you want to deploy.
  const deployer = new Deployer(hre, wallet);
  const artifact = await deployer.loadArtifact("Greeter");

  // Estimate contract deployment fee
  const greeting = "Hi there!";
  const deploymentFee = await deployer.estimateDeployFee(artifact, [greeting]);

  // ⚠️ OPTIONAL: You can skip this block if your account already has funds in L2
  // Deposit funds to L2
  // const depositHandle = await deployer.zkWallet.deposit({
  //   to: deployer.zkWallet.address,
  //   token: utils.ETH_ADDRESS,
  //   amount: deploymentFee.mul(2),
  // });
  // // Wait until the deposit is processed on zkSync
  // await depositHandle.wait();

  // Deploy this contract. The returned object will be of a `Contract` type, similarly to ones in `ethers`.
  // `greeting` is an argument for contract constructor.
  const parsedFee = ethers.utils.formatEther(deploymentFee.toString());
  console.log(`The deployment is estimated to cost ${parsedFee} ETH`);

  const greeterContract = await deployer.deploy(artifact, [greeting]);

  //obtain the Constructor Arguments
  console.log("Constructor args:" + greeterContract.interface.encodeDeploy([greeting]));

  // Show the contract info.
  const contractAddress = greeterContract.address;
  console.log(`${artifact.contractName} was deployed to ${contractAddress}`);

  // verify contract for testnet & mainnet
  if (process.env.NODE_ENV != "test") {
    // Contract MUST be fully qualified name (e.g. path/sourceName:contractName)
    const contractFullyQualifedName = "contracts/Greeter.sol:Greeter";

    // Verify contract programmatically
    const verificationId = await hre.run("verify:verify", {
      address: contractAddress,
      contract: contractFullyQualifedName,
      constructorArguments: [greeting],
      bytecode: artifact.bytecode,
    });
  } else {
    console.log(`Contract not verified, deployed locally.`);
  }
}
```

#### 4：使用以下命令运行部署脚本：

```shell
yarn hardhat deploy-zksync --script deploy-greeter.ts --network zkSyncTestnet
```
















































