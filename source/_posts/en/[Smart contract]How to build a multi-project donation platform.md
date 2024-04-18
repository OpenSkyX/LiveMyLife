---
title: Smart contract | How to build a multi-project donation platform
catalog: true
date: 2023-08-4 2:20:17
subtitle: 智能合约 | 如何做一个多项目捐款平台
header-img: /img/header_img/lml_bg.jpg
tags:
- Web3-Develop
categories:
- Web3-Develop
---

## 快速开始

     项目介绍，管理员创建项目，由捐款人进行捐款ETH，捐款后自动获得捐款证明的NFT凭证，用于接收后期的项目分红。


## 创建 WaterDrop Demo

```shell
mkdir WaterDrop 
cd WaterDrop
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

## 在contract目录下创建 WaterDrop.sol

```shell
touch WaterDrop.sol
vim WaterDrop.sol
```

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";


contract MyNFT is ERC721Enumerable, Ownable {
    uint256 public tokenIdCounter;
    constructor(string memory name, string memory symbol) ERC721(name, symbol) {
        tokenIdCounter = 1;
    }
    function mint(address to) external onlyOwner {
        uint256 tokenId = tokenIdCounter;
        _mint(to, tokenId);
        tokenIdCounter++;
    }
    function _baseURI() internal pure override returns (string memory) {
        return "https://example.com/token/";
    }
}

contract WaterDrop {
    MyNFT  Nft;
    address public owner;
    uint public projectTax;
    uint public projectCount;
    uint public balance;
    statsStruct public stats;
    projectStruct[] projects;

    mapping(address => projectStruct[]) projectsOf;
    mapping(uint => backerStruct[]) backersOf;
    mapping(uint => bool) public projectExist;

    enum statusEnum {
        OPEN,
        APPROVED,
        REVERTED,
        DELETED,
        PAIDOUT
    }

    struct statsStruct {
        uint totalProjects;
        uint totalBacking;
        uint totalDonations;
    }

    struct backerStruct {
        address owner;
        uint contribution;
        uint timestamp;
        bool refunded;
    }

    struct projectStruct {
        uint id;
        address owner;
        string title;
        string description;
        string imageURL;
        uint cost;
        uint raised;
        uint timestamp;
        uint expiresAt;
        uint backers;
        statusEnum status;
    }

    modifier ownerOnly(){
        require(msg.sender == owner, "Owner reserved only");
        _;
    }

    event Action (
        uint256 id,
        string actionType,
        address indexed executor,
        uint256 timestamp
    );

    constructor(uint _projectTax) {
        owner = msg.sender;
        projectTax = _projectTax;
        //创建NFT
        Nft = new MyNFT("XXX","X");
    }
    //创建捐款项目
    function createProject(
        string memory title,
        string memory description,
        string memory imageURL,
        uint cost,
        uint expiresAt
    ) public returns (bool) {
        require(bytes(title).length > 0, "Title cannot be empty");
        require(bytes(description).length > 0, "Description cannot be empty");
        require(bytes(imageURL).length > 0, "ImageURL cannot be empty");
        require(cost > 0 ether, "Cost cannot be zero");

        projectStruct memory project;
        project.id = projectCount;
        project.owner = msg.sender;
        project.title = title;
        project.description = description;
        project.imageURL = imageURL;
        project.cost = cost;
        project.timestamp = block.timestamp;
        project.expiresAt = expiresAt;

        projects.push(project);
        projectExist[projectCount] = true;
        projectsOf[msg.sender].push(project);
        stats.totalProjects += 1;

        emit Action (
            projectCount++,
            "PROJECT CREATED",
            msg.sender,
            block.timestamp
        );
        return true;
    }
    
    //更新捐款项目
    function updateProject(
        uint id,
        string memory title,
        string memory description,
        string memory imageURL,
        uint expiresAt
    ) public returns (bool) {
        require(msg.sender == projects[id].owner, "Unauthorized Entity");
        require(bytes(title).length > 0, "Title cannot be empty");
        require(bytes(description).length > 0, "Description cannot be empty");
        require(bytes(imageURL).length > 0, "ImageURL cannot be empty");

        projects[id].title = title;
        projects[id].description = description;
        projects[id].imageURL = imageURL;
        projects[id].expiresAt = expiresAt;

        emit Action (
            id,
            "PROJECT UPDATED",
            msg.sender,
            block.timestamp
        );

        return true;
    }
    
    //删除捐款项目
    function deleteProject(uint id) public returns (bool) {
        require(projects[id].status == statusEnum.OPEN, "Project no longer opened");
        require(msg.sender == projects[id].owner, "Unauthorized Entity");

        projects[id].status = statusEnum.DELETED;
        performRefund(id);

        emit Action (
            id,
            "PROJECT DELETED",
            msg.sender,
            block.timestamp
        );

        return true;
    }
    
    
    function performRefund(uint id) internal {
        for(uint i = 0; i < backersOf[id].length; i++) {
            address _owner = backersOf[id][i].owner;
            uint _contribution = backersOf[id][i].contribution;

            backersOf[id][i].refunded = true;
            backersOf[id][i].timestamp = block.timestamp;
            payTo(_owner, _contribution);

            stats.totalBacking -= 1;
            stats.totalDonations -= _contribution;
        }
    }
    
    //进行捐款
    function backProject(uint id) public payable returns (bool) {
        require(msg.value > 0 ether, "Ether must be greater than zero");
        require(projectExist[id], "Project not found");
        require(projects[id].status == statusEnum.OPEN, "Project no longer opened");

        stats.totalBacking += 1;
        stats.totalDonations += msg.value;
        projects[id].raised += msg.value;
        projects[id].backers += 1;

        backersOf[id].push(
            backerStruct(
                msg.sender,
                msg.value,
                block.timestamp,
                false
            )
        );

        emit Action (
            id,
            "PROJECT BACKED",
            msg.sender,
            block.timestamp
        );

        if(projects[id].raised >= projects[id].cost) {
            projects[id].status = statusEnum.APPROVED;
            balance += projects[id].raised;
            performPayout(id);
            Nft.mint(msg.sender);
            return true;
        }

        if(block.timestamp >= projects[id].expiresAt) {
            projects[id].status = statusEnum.REVERTED;
            performRefund(id);
            return true;
        }

        return true;
    }
    
    //付款
    function performPayout(uint id) internal {
        uint raised = projects[id].raised;
        uint tax = (raised * projectTax) / 100;

        projects[id].status = statusEnum.PAIDOUT;

        payTo(projects[id].owner, (raised - tax));
        payTo(owner, tax);

        balance -= projects[id].raised;

        emit Action (
            id,
            "PROJECT PAID OUT",
            msg.sender,
            block.timestamp
        );
    }

    function requestRefund(uint id) public returns (bool) {
        require(
            projects[id].status != statusEnum.REVERTED ||
            projects[id].status != statusEnum.DELETED,
            "Project not marked as revert or delete"
        );

        projects[id].status = statusEnum.REVERTED;
        performRefund(id);
        return true;
    }

    function payOutProject(uint id) public returns (bool) {
        require(projects[id].status == statusEnum.APPROVED, "Project not APPROVED");
        require(
            msg.sender == projects[id].owner ||
            msg.sender == owner,
            "Unauthorized Entity"
        );

        performPayout(id);
        return true;
    }

    function changeTax(uint _taxPct) public ownerOnly {
        projectTax = _taxPct;
    }

    function getProject(uint id) public view returns (projectStruct memory) {
        require(projectExist[id], "Project not found");

        return projects[id];
    }

    function getProjects() public view returns (projectStruct[] memory) {
        return projects;
    }

    function getBackers(uint id) public view returns (backerStruct[] memory) {
        return backersOf[id];
    }

    function payTo(address to, uint256 amount) internal {
        (bool success, ) = payable(to).call{value: amount}("");
        require(success);
    }

    function getNftAddress() public view returns(MyNFT){
        return Nft;
    }
    
    //test 
    function mintNft() public{
        Nft.mint(msg.sender);
    }
}
```

## 在scripts 文件夹下创建WaterDrop-deploy.js

```javascript
//hardhat库使用ethers组件与区块链进行交互
//hardhat库使用ethers组件与区块链进行交互
const { ethers } = require("hardhat");
async function main() {
    const Genesis = await  ethers.getContractFactory('WaterDrop')
    const genesis = await Genesis.deploy(1);
    console.log('WaterDrop',await genesis.target)
    console.log("nft:",await genesis.getNftAddress())
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
yarn hardhat run ./scripts/WaterDrop-deploy.js
```

## 在test文件夹下创建Swap-test.js测试文件

```javascript
const { expect } = require('chai') //断言模块
const { ethers} = require('hardhat') //安全帽模块


describe('WaterDrop 合约测试', () => {

    before(async function () {
        //获取合约工厂对象
        this.WaterDrop = await ethers.getContractFactory('WaterDrop')
        //通过合约工厂部署合约
        this.waterDrop = await this.WaterDrop.deploy(1)
    })


    it('create project', async function () {
        const result = await this.waterDrop.createProject("test solidity", "test", "http://test.dev", 1, 2);
        console.log(result.hash)
        // expect(result).to.be.true;
    });

    it('get project', async function () {
        /*
        * uint id;
        address owner;
        string title;
        string description;
        string imageURL;
        uint cost;
        uint raised;
        uint timestamp;
        uint expiresAt;
        uint backers;
        statusEnum status;*/
        const projects = await this.waterDrop.getProjects();
        for (let i = 0; i < projects.length; i++) {
            console.log("id:", projects[i].id, "owner:", projects[i].owner, "title:", projects[i].title, "description:", projects[i].description,
                "imageURL:", projects[i].imageURL, "cost:", projects[i].cost, "raised:", projects[i].raised,
                "timestamp:", projects[i].timestamp, "expiresAt:", projects[i].expiresAt, "backers:", projects[i].backers, "status:", projects[i].status)
        }
        this.projects = projects;
    });

    it('update project ', async function () {
        /*
        uint id,
        string memory title,
        string memory description,
        string memory imageURL,
        uint expiresAt
        * */
        const project = this.projects[0];
        console.log(project.id, project.title, project.description, project.imageURL, project.expiresAt)
        const transaction = await this.waterDrop.updateProject(project.id, project.title, project.description, project.imageURL, project.expiresAt);
        console.log(transaction.hash);
    });

    it('backProject', async function () {
        // Get the signer object for the address
        const balance = await ethers.provider.getBalance("0x1a8e3EA28dC782DD37357D02B68dfe5eAff05C7B");
        console.log("balance:", balance)
        const amountToSend = ethers.parseEther('1.0')
        const transaction = await this.waterDrop.backProject(this.projects[0].id, {value: amountToSend});
        console.log(transaction.hash)
    });
})
```

## 执行测试任务 (默认是hardhat的环境，需要更改网络请添加  --network  NETWORKNAME 配置文件里的网络名称)

```shell
yarn hardhat test ./test/WaterDrop-test.js --network local
```

## 测试任务执行完成后会显示：

```text
  WaterDrop 合约测试
0xad610b6d6aaf7f8ebb96d3d2bde909750be53af7e25cf803e0312b4bc56a112a
    √ create project (1316ms)
id: 0n owner: 0xddCfcb8Fd3E86569BcFa2d85Ab6e5F5621977667 title: test solidity description: test imageURL: http://test.dev cost: 1n rais
ed: 0n timestamp: 1691087429n expiresAt: 2n backers: 0n status: 0n
    √ get project (89ms)
0n test solidity test http://test.dev 2n
0x8c53be04ac9c42db1f01f58009885e05ecf3c3935f913576d5211045ae9c0ef9
    √ update project  (655ms)
balance: 0n
0xaaa514936c71416b58b729c13ed97469996b369a0ff2f4ee0a97974e86290e7d
    √ backProject (1364ms)


  4 passing (4s)

Done in 6.56s.
```

## 至此，一个多项目捐款平台合约开发完毕！
