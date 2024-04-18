---
title: Use hardhat to build dapp and develop smart contracts and deployment
catalog: true
date: 2023-07-29 02:34:17
subtitle: Use hardhat for smart contract development...
header-img: /img/header_img/lml_bg.jpg
tags:
- Web3-Develop
categories:
- Web3-Develop
---

## Quick Start

## Create Demo
```shell
mkdir demo 
cd demo
npm init  #Confirm generating package.json
```

## Use the yarn command to install hardhat project package
```shell
yarn add hardhat
```

## Install the openzeppelin library
```shell
yarn add @openzeppelin/contracts 
yarn add @openzeppelin/hardhat-upgrades
```

## Install the test plugin
```shell
yarn add -D chai
```

## Install the ethers

```shell
yarn add -D ethers
```

## Install the ethereum-waffle

```shell
npm install --save-dev ethereum-waffle
```

## After the above is successfully installed,you can write smart contract. View hardhat operation commands by executing npx hardhat --help
```shell
npx hardhat --help

全局选项
--config              一个安全帽配置文件.
--emoji               在信息中使用表情符号.
--help                显示此消息，如果提供了任务名称，则显示任务的帮助
--max-memory          安全帽可以使用的最大内存量
--network             要连接到的网络.
--show-stack-traces   显示堆栈跟踪.
--tsconfig            保留的安全帽参数.
--verbose             支持安全帽详细日志记录
--version             版本显示安全帽的版本
可用任务
check         检查你需要的任何东西
clean         清除缓存并删除所有工件
compile       编译编译整个项目，构建所有工件
console       控制台打开安全帽控制台
flatten       展平并打印合同及其依赖项
help          帮助
node          在Hardhat网络上启动JSON-RPC服务器
run           在编译项目后运行用户定义的脚本
test          运行摩卡测试
```

## Run npx hardhat command(select Create an empty hardhat config.js)

```shell
npx hardhat
Welcome to Hardhat v2.6.8

? What do you want to do? …
Create a basic sample project
Create an advanced sample project
Create an advanced sample project that uses TypeScript
▸ Create an empty hardhat.config.js
Quit
```

## Open config.js and change setting  

```javascript
//hardhat项目依赖组件
require("@nomiclabs/hardhat-waffle");
require('@openzeppelin/hardhat-upgrades');

//hardhat项目配置项
module.exports = {
    solidity: "0.8.4", //使用的sodity库的版本
    networks: {
        local: {
            url: 'http://127.0.0.1:8545', //本地RPC地址
            //本地区块链账户地址(需要启动运行npx hardhat node命令开启本地开发环境的区块链)
            //这些账户地址和秘钥每次重启区块链都是相同的,并且数据会重置
            accounts: [
                // 0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266 (第一个账户地址及秘钥)
                '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
                // 0x70997970c51812dc3a010c7d01b50e0d17dc79c8 (第二个账户地址及秘钥)
                '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d',
                // 0x3c44cdddb6a900fa2b585dd299e03d12fa4293bc (三个账户地址及秘钥)
                '0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a',
                // 0x90f79bf6eb2c4f870365e785982e1f101e93b906 (第四个个账户地址及秘钥)
                '0x7c852118294e51e653712a81e05800f419141751be58f605c371e15141b007a6',
                // 0x15d34aaf54267db7d7c367839aaf71a00a2c6a65 (第五个账户地址及秘钥)
                '0x47e179ec197488593b187f80a00eb0da91f1b9d0b13f8733639f19c30a34926a',
            ]
        }
    }
};
```

## Create a contracts directory under the demo directory for writing smart contract 

```shell
mkdir contracts
```

## Create a test directory under the demo directory for writing unit test

```shell
mkdir test
```

## Create a scripts directory under the demo directory for  writing script

```shell
mkdir scripts
```

## Create a Demo.sol file under the contracts for writing smart contract

```shell
touch Demo.sol
vim Demo.sol
```

## Edit Demo.sol first write simplest a smart contract

``` solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

contract Demo {
uint256  public score = 0;
}
```

## Edit demo-deploy.js deploy script under scripts directory

```javascript
$ touch demo-deploy.js
$ vim demo-deploy.js

//hardhat库使用ethers组件与区块链进行交互
const { ethers } = require("hardhat");

//main function
async function main() {
    const Demo = await ethers.getContractFactory("Demo");
    const demo = await Demo.deploy()
    console.log(demo.address)
}

//execute function
main().then(() => process.exit(0)).catch(error => {
    console.error(error);
    process.exit(1);
});
```

## Deploy Smart Contract （method one）

```shell
npx hardhat run scripts/demo-deploy.js
```

## Deploy Smart Contract （method two） setting rpc network

```shell
npx hardhat run scripts/demo-deploy.js --network local
```

## interact with the Demo contract for testing

### 1: Create a demo.test.js under test directory
```javascript
touch demo.test.js
$ vim demo.test.js

const { expect } = require('chai') //断言模块
const { ethers} = require('hardhat') //安全帽模块

describe('Demo合约测试', () => {

    /**

     * 测试执行前的钩子函数
     */
    before(async function () {
        //获取合约工厂对象
        this.Demo = await ethers.getContractFactory('Demo')


        //通过合约工厂部署合约
        this.demo = await this.Demo.deploy()

    })

    /**

     * 获取score状态测试
     */
    it('测试score等于0', async function () {
        const score = await this.demo.score()
        expect(score.toString()).to.be.equal('0')
    })

})
```

## Execute Test
```shell
npx hardhat test test/demo.test.js --network local
```

### So far,a simple process of smart contract writing,deployment,and testing has been completed,but this is just a start.
### Now our smart contract only has the function of reading score,now we need to add functions to the smart contract

### Next, we have another demo ,which is to add a setScore method,which can change the value of score

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

//Demo合约
contract Demo {
  uint256  public score = 0;
  //修改score的状态
  function setScore(uint256 newScore) public returns(bool) {
    score = newScore;
    return true;
  }
}
```

## Added setScore contract method adds test cases

```javascript
const { expect } = require('chai') //断言模块
const { ethers} = require('hardhat') //安全帽模块

describe('Demo合约测试', () => {

    /**

     * 测试执行前的钩子函数
     */
    before(async () => {
        //获取合约工厂对象
        this.Demo = await ethers.getContractFactory('Demo')

        //通过合约工厂部署合约
        this.demo = await this.Demo.deploy()

    });

    /**

     * get score test
     */
    it('测试score等于0', async () => {
        const score = await this.demo.score()
        expect(score.toString()).to.be.equal('0')
    })

    /**

     * modify score test
     */
    it('修改score等于100', async () => {
        await this.demo.setScore(100)
        const score = await this.demo.score()
        expect(score.toString()).to.be.equal('100')
    })

})
```

## Execute new test cases

```shell
npx hardhat test test/demo.test.js --network local
```

## Now we can develop the contract by writing the Demo.sol file
## Write upgradable smart contract
### create a demo-upgradeable-deploy.js under scripts directory
```javascript
const { ethers, upgrades } = require("hardhat")
//main function 
async function main() {
    const Demo = await ethers.getContractFactory('Demo')
    console.log('Deploying Demo...')
    const demo = await upgrades.deployProxy(Demo, [300], { initializer:  setScore' })
        await demo.deployed()
        console.log('Demo to:', demo.address)
    }

//执行可升级合约部署
    main().then(() => process.exit(0)).catch(error => {
        console.error(error)
        process.exit(1)
    });
```

###  Execute upgradable deploy contract
```shell
npx hardhat run scripts/demo-upgradeable-deploy.js --network local

Deploying Demo...
Demo to: 0x959922bE3CAee4b8Cd9a407cc3ac1C251C2007B1
```

    After the devlopment is complete, we need to save our contract address,witch will be the user-oriented contract interaction address in the future.
    Every time upgrade the contract,need this address as the proxy contract address to bind with the newly upgraded contract

### Start writing the upgrade contract DemoV2.sol,Added the increment method to the upgraded contract,and the score value is increased by 1 every time it's called.

```solidity
// contracts/DemoV2.sol
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

//Demo合约
contract DemoV2 {
    uint256  public score;

    //修改score的状态
    function setScore(uint256 newScore) public {
        score = newScore;
    }

    //分值加1
    function increment() public {
        score = score + 1;
    }

}
```

### Edit deploy upgradable contract script

```javascript
//安全帽模块
const { ethers, upgrades  } = require("hardhat");
//主函数
async function main() {
    const upgradeContractName = 'DemoV2' //升级合约的名称
    const proxyContractAddress = '0x959922bE3CAee4b8Cd9a407cc3ac1C251C2007B1' //代理合约的名称
    const DemoUpgrade = await ethers.getContractFactory(upgradeContractName)
    console.log('Upgrading Demo...')
    await upgrades.upgradeProxy(proxyContractAddress, DemoUpgrade)
    console.log('Demo upgraded')
}
//升级合约
main().then(() => process.exit(0)).catch(error => {
    console.error(error)
    process.exit(1)
})
```

### Execute contract upgraded

```shell
npx hardhat run scripts/demo-upgrade.js --network local
```

### Verify the upgrade contract through the hardhat console.

```shell
npx hardhat console --network local
```

### Upgrade contract  write test cases

```javascript
//demo-v2.test.js
const { expect } = require('chai') //断言模块 
const { ethers, upgrades} = require('hardhat') //安全帽模块

describe('DemoV2升级合约测试', () => {

    /**
     * 测试执行前的钩子函数
     */
    before(async () => {
        //从工厂获取合约
        const DemoContract = await ethers.getContractFactory('Demo')
        //部署可升级代理合约
        const demoProxy = await upgrades.deployProxy(DemoContract, [300], { initializer: 'setScore' })
        //部署
        this.demo = await demoProxy.deployed()
    });

    /**
     * 获取score状态测试
     */
    it('测试score等于300', async () => {
        const score = await this.demo.score() //读取score 
        expect(score.toString()).to.be.equal('300') //断言结果为300
    })

    /**
     * 修改score状态测试
     */
    it('修改score等于100', async () => {
        await this.demo.setScore(100) //设置score为100
        const score = await this.demo.score() //读取score  
        expect(score.toString()).to.be.equal('100') //断言结果为100
    })

    /**
     * 计数器加1测试
     */
    it('increment计数器测试', async () => {
        const upgradeContractName = 'DemoV2' //升级合约的名称
        const proxyContractAddress = this.demo.address //代理合约的名称
        const DemoUpgrade = await ethers.getContractFactory(upgradeContractName) //工厂合约
        const demoV2 = await upgrades.upgradeProxy(proxyContractAddress, DemoUpgrade) //升级合约
        await this.demo.setScore(800) //设置score为800
        await demoV2.increment() //计数器+1
        const newScore = (await this.demo.score()).toString() //获取存储的新值
        expect(newScore).to.be.equal('801') //断言结果为801
    })
})
```









