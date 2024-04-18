---
title: Smart-Contract | How to develop a simple token swap smart contract
catalog: true
date: 2023-08-3 12:34:17
subtitle: 智能合约 | 如何开发简单的代币交换智能合约
header-img: /img/header_img/lml_bg.jpg
tags:
- Web3-Develop
categories:
- Web3-Develop
---

## 快速开始

## 创建 swap Demo

```shell
mkdir swap 
cd swap
npm init  #默认生成 package.json
```

## 使用yarn命令安装hardhat项目包

```shell
yarn add hardhat
```

## 安装openzeppelin依赖

```shell
yarn add @openzeppelin/contracts 
yarn add @openzeppelin/hardhat-upgrades
```

## 安装测试断言插件

```shell
yarn add -D chai
```

## 安装ethers

```shell
yarn add -D ethers
```

## 安装 ethereum-waffle

```shell
npm install --save-dev ethereum-waffle
```

## 运行 npx hardhat 并选择（▸ Create an empty hardhat.config.js）创建hardhat.config.js文件

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

## 打开 config.js 

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
            ]
        }
    }
};
```

## 创建contracts文件夹

```shell
mkdir contracts
```

## 创建test文件夹

```shell
mkdir test
```

## 创建scripts文件夹

```shell
mkdir scripts
```

## 在contract目录下创建 swap.sol

```shell
touch Swap.sol
vim Swap.sol
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IERC20 {
    // 获取合约地址
    function getAddress() external  view returns (address);
    // 获取代币发行总量
    function totalSupply() external  view returns (uint256);
    // 根据地址获取代币的余额
    function balanceOf(address account) external  view returns (uint256);
    // 代理可转移的代币数量
    function allowance(address owner, address supender) external  view returns (uint256);

    // 转账
    function transfer(address recipient, uint256 amount) external  returns (bool);
    // 设置代理能转账的金额
    function approve(address owner, address spender, uint256 amount) external  returns (bool);
    // 转账
    function transferFrom(address sender, address recipient, uint256 amount) external  returns (bool);

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}

// 代币实现
contract ERC20Basic is IERC20 {
    string public constant name = "X"; // 代币名称
    string public constant symbol = "X"; // 代币简称
    uint8 public constant decimals = 18;

    mapping(address => uint256) balances; // 地址对应的余额数量
    mapping(address => mapping(address => uint256)) allowedBalence; // 代理商能处理的代币数量

    uint256 totalSupply_ = 10 ether; // 发行数量，ether指的是单位，类似吨，也可以使用8个0


    constructor () {
        balances[msg.sender] = totalSupply_; // 将代币分发给创建者
    }

    // 获取合约地址
    function getAddress() external override view returns (address){
        return address(this); // 当前合约的地址
    }
    // 获取代币发行总量
    function totalSupply() external override view returns (uint256){
        return totalSupply_;
    }

    // 根据地址获取代币的余额
    function balanceOf(address tokenOwner) public override view returns (uint256){
        return balances[tokenOwner]; // 根据地址获取余额
    }

    // 转账
    function transfer(address receiver, uint256 amount) public override returns (bool){
        require(amount <= balances[msg.sender]);
        balances[msg.sender] = balances[msg.sender]-amount;
        balances[receiver] = balances[receiver]+amount;
        emit Transfer(msg.sender, receiver, amount);
        return true;
    }

    // 设置代理能转账的金额
    function approve(address owner, address delegate, uint256 amount) external override returns (bool){
        allowedBalence[owner][delegate] = amount;
        emit Approval(owner, delegate, amount);
        return true;
    }

    // 代理可转移的代币数量
    function allowance(address owner, address delegate) external override view returns (uint256){
        return allowedBalence[owner][delegate];
    }

    // 转账
    function transferFrom(address owner, address buyer, uint256 amount) external override returns (bool){
        require(amount <= balances[owner]);
        require(amount <= allowedBalence[owner][msg.sender]);

        balances[owner] = balances[owner]-amount;
        allowedBalence[owner][buyer] = allowedBalence[owner][buyer]-amount;
        balances[buyer] = balances[buyer]+amount;
        emit Transfer(owner, buyer, amount);
        return true;
    }
}

contract Swap {

    IERC20 public token;

    event Buy(uint256 amount);
    event Sell(uint256 amount);
    event Withdraw(address _addrss,uint256 _amount);

    constructor(){
        token = new ERC20Basic();
    }

    function buy()public payable {
        uint256 inAmount = msg.value;
        uint256 dexBalance = token.balanceOf(address(this));
        require(inAmount > 0,'You need send some ether!');
        require(inAmount <= dexBalance,'Not enough tokens in the reserve');

        uint256 outAmount = inAmount * 20;
        token.transfer(msg.sender,outAmount);
        emit Sell(outAmount);
    }

    function approve(uint256 amount) public returns(bool){
        return token.approve(msg.sender,address(this),amount);
    }
    
    
    function sell(uint256 amount) public {
        /// 在这里只简单演示卖掉erc20代币，退回合约中的eth
        require(amount > 0,'You need to sell at least some tokens');
        uint256 allowance = token.allowance(msg.sender,address(this));
        require(amount <= allowance,'check the token allowance');
        ///发送ERC20代币给合约
        token.transferFrom(msg.sender,address(this),amount);
        uint256 withdrawAmount = address(this).balance;
        /// 退回eth
        payable(msg.sender).call{value: withdrawAmount}("");
        ///触发事件
        emit Withdraw(address(msg.sender),withdrawAmount);
        emit Sell(amount);
    }

    function getDexBalance() public view returns(uint256){
        return token.balanceOf(address(this));
    }

    function getOwnerBalance() public view returns(uint256){
        return token.balanceOf(msg.sender);
    }

    function getTotalSupply() public view returns(uint256){
        return token.totalSupply();
    }

    function getAllowance() public view returns(uint256){
        return token.allowance(msg.sender,address(this));
    }

    function getEthBalance() public view returns(uint256){
        return address(this).balance ;
    }
}
```

## 在scripts 文件夹下创建Swap-deploy.js

```javascript
//hardhat库使用ethers组件与区块链进行交互
const { ethers } = require("hardhat");

async function main() {
    const XDEX = await  ethers.getContractFactory('Swap')
    const xdex = await XDEX.deploy();
    console.log('XDex',xdex.target)
}

//执行部署
main().then(() => process.exit(0)).catch(error => {
    console.error(error);
    process.exit(1);
});
```

## 执行编译命令

```shell
yarn hardhat compile
```
## 执行部署命令

```shell
yarn hardhat run ./scripts/Swap-deploy.js
```

## 执行完成后会显示

```shell
XDex 0x2E3966dF4c31C1592337D183d166c856594A70A5
Done in 3.70s.
```

## 在test文件夹下创建Swap-test.js测试文件

```javascript
const { expect } = require('chai') //断言模块
const { ethers} = require('hardhat') //安全帽模块

describe('Dex 合约测试', () => {
    before(async function () {
        //获取合约工厂对象
        this.XDex = await ethers.getContractFactory('XDex')
        //通过合约工厂部署合约
        this.dex = await this.XDex.deploy()
    })

    it('get contract address',async function () {
       const address = await this.dex.target
        console.log('get contract address:',address)
    });

    it('Dex buy token',async function () {
        const amountToSend = ethers.parseEther("0.1");
        console.log(amountToSend)
        const transaction = await this.dex.buy({ value: amountToSend })
        // 确认交易成功
        await transaction.wait();
        //获取以太坊余额
        const address = await this.dex.target
        const contractBalance = await ethers.provider.getBalance(address)
        // 断言合约余额为 amountToSend ETH
        expect(contractBalance).to.equal(amountToSend);
    });

    it('dex Sell tokens',async function () {
        const amountToSend = ethers.parseEther("2");
        const x = await this.dex.approve(amountToSend);
        console.log("hash:",x.hash)
        const getAllowance = await this.dex.getAllowance();
        console.log("getAllowance:",getAllowance)
        await this.dex.sell(amountToSend);
    });

    it('getDexBalance',async function () {
        const dexBalance = await this.dex.getDexBalance();
        console.log("getDexBalance:",dexBalance);
    });

    it('getOwnerBalance ',async function () {
        const getOwnerBalance  = await this.dex.getOwnerBalance();
        console.log("OwnerBalance:",getOwnerBalance);
    });

    it('getTotalSupply',async function () {
        const getTotalSupply = this.dex.getTotalSupply();
        console.log("getTotalSupply:",getTotalSupply)
    });

    it('getEthBalance:',async function () {
       const getEthBalance = await this.dex.getEthBalance();
       console.log("getEthBalance:",getEthBalance)
    });
})
```
## 执行测试任务 (默认是hardhat的环境，需要更改网络请添加  --network  NETWORKNAME 配置文件里的网络名称)

```shell
yarn hardhat test 
```

## 测试任务执行完成后会显示：

```text
  Swap 合约测试
get contract address: 0x5FbDB2315678afecb367f032d93F642f64180aa3
    √ get contract address
100000000000000000n
    √ Dex buy token (55ms)
hash: 0x6130311d6d209dfd69a9113993be4f45781ff96cd4bf8ba7c6cabb32b09d31a7
getAllowance: 2000000000000000000n
    √ dex Sell tokens (72ms)
getDexBalance: 10000000000000000000n
    √ getDexBalance
OwnerBalance: 0n
    √ getOwnerBalance 
getTotalSupply: Promise { <pending> }
    √ getTotalSupply
getEthBalance: 0n
    √ getEthBalance:


  7 passing (5s)

Done in 6.53s.

```
## 至此，一个拥有简单的代币合约兑换功能的合约开发完毕！
