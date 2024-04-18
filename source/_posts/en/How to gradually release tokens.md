---
title: How to gradually release tokens
catalog: true
date: 2023-08-16 9:40:17
subtitle: 如何逐步释放token 
header-img: /img/header_img/lml_bg.jpg
tags:
- Web3-Develop
categories:
- Web3-Develop
---

## Quick Start  | 释放规则
> 所有STRAC处于锁定状态，不可转账，不可卖出，不可添加流动性。
释放
> 
> 根据持币地址锁定的STRAC数量，每24小时释放万分之六。其中，每小时会进行一次解锁释放，即从地址持有STRAC的一刻开始计算时间，每小时解锁释放的STRAC数量为：该地址锁定STRAC数量*6/10000/24；
> 
> 用户仅可对地址上已解锁释放出来的STRAC数量进行操作，包括转账、卖出、添加流动性。同一地址在一个小时之内，只可对已解锁释放出来的STRAC进行一次操作。包括但不限于一个小时之内，同一地址只可对其他地址进行一次STRAC转账，或者卖出，或者添加流动性；
> 
> 当用户地址接收到其他地址转入STRAC时，以及用户在交易所购买所得的STRAC时，以及用户移除流动性所得的STRAC时，STRAC都会处于锁定状态，需要按照每24小时万分之六进行解锁释放；
> 
> 一小时释放周期之内，若用户收到多笔来自不同地址的STRAC转账时，会在一小时到期之后，统一进行解锁释放；
> 
> 若用户地址在获得STRAC并且首次解锁释放之后，该地址未触发合约进行其他操作，如未对已解锁释放出来的STRAC进行操作，以及未收到其他地址对该地址转入的STRAC（包括接收转账、交易所买入/卖出、添加/移除流动性），则始终按照该地址上总锁定STRAC数量，进行24小时万分之六释放；
> 
> 若用户地址在获得STRAC并且首次解锁释放之后，该地址触发合约进行其他操作，如对已解锁释放出来的STRAC进行操作，或者收到其他地址对该地址转入的STRAC（包括接收转账、交易所买入/卖出、添加/移除流动性），则需要按照该地址上总锁定STRAC数量减去已解锁释放出来的STRAC数量，进行24小时万分之六释放；
> 
> 买入0.3%手续费进入黑洞；
卖出3%手续费进入自动卖出合约，每天50%卖出换成ETH，与另外剩余50%添加进底池，LP自动转入黑洞。
>

### 开发环境

> remix 

### 直接上代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/security/Pausable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "hardhat/console.sol";

contract XERC20 is ERC20, ERC20Burnable, Pausable, Ownable {
    uint256 public amountAllowed = 100; // Number of claims per time
    mapping(address => bool) requestAddress; // received

    address public pairAddress = address(0); //Test

    struct DataRelease {
        uint256 lastUpdateTime; //最后释放时间
        uint256 releaseAmount; //已释放余额
    }

    mapping(address => DataRelease) public datas;
    uint256 public releaseInterval = 60 ;  //1分钟

    event MintFaucetToken(address _receive, uint256 _amount);

    constructor() ERC20("XMax", "XMX") {}

    function pause() public onlyOwner {
        _pause();
    }

    function unpause() public onlyOwner {
        _unpause();
    }
    
    //用来测试空投领取，故没有添加权限，
    function mint(address to, uint256 amount) public {
        calculateRelease(to);
        _mint(to, amount);
        datas[msg.sender].lastUpdateTime = block.timestamp;
    }

    function _transfer(
        address from,
        address to,
        uint256 amount
    ) internal override {
        console.log("override _transfer");
        // 计算已释放余额
        calculateRelease(from);
        //判断转账余额是否大于已释放余额
        require(
            datas[from].releaseAmount >= amount,
            "Insufficient balance to release"
        );

        //TODO 手续费、白名单  

        releaseFrom(from, amount);
        releaseTo(to);
        super._transfer(from, to, amount);
    }

    //from -- 
    function releaseFrom(address _from, uint256 _amount) internal {
        datas[_from].releaseAmount  -= _amount;          // 从已释放余额中减去转账除去的数量
    }
    // to ++  
    function releaseTo(address _to) internal {
        calculateRelease(_to); //先计算到当前时间TO地址需要释放两
        datas[_to].lastUpdateTime = block.timestamp;  // 更新时间
    }

    function calculateRelease(address _address)
        internal
    {
        uint256 elapsedTime = block.timestamp - datas[_address].lastUpdateTime;
        console.log("elapsedTime:",elapsedTime);
        uint256 elapsedHours = elapsedTime / releaseInterval;
        console.log("elapsedHours:",elapsedHours);
        if(elapsedHours == 0 || balanceOf(_address) == 0 )  return;  //释放时间为0或者余额为0 ，退出计算
        
        //计算每小时释放多少币   待解锁数量 = balance（实际数量）- releaseAmount （已释放数量）
        uint256 waitRelease = balanceOf(_address) - datas[_address].releaseAmount;
        // 待释放数量为0退出
        if(waitRelease == 0) return;  
        uint256 releasePerInterval = (waitRelease * 6) / 10000 / 24;
        //计算当前时间释放总量
        
        //TODO 此时会产生一定的时间差，在1小时内，若需要，需要在当前时间减去相差的时间差。 比如：此时过了1:50分，但是计算的只有1小时，那50分钟是丢失的，此时将时间更新为当前时间，是不合理的。
        uint256 releaseAmount = releasePerInterval * elapsedHours;
        datas[_address].releaseAmount += releaseAmount;
        datas[_address].lastUpdateTime = block.timestamp;
    }   


    //查询当前总共释放金额，但会刷新时间，如果不需刷新时间，去掉计算代码
    function getReleaseAmount(address _address) external  returns(uint256){
        calculateRelease(_address);
        return datas[_address].releaseAmount;
    }

    function _beforeTokenTransfer(
        address from,
        address to,
        uint256 amount
    ) internal override whenNotPaused {
        super._beforeTokenTransfer(from, to, amount);
    }

    //领取测试币
    function requestTokens() external {
        mint(msg.sender, amountAllowed * 10**18);
        emit MintFaucetToken(msg.sender, amountAllowed * 10**18);
    }
}
```

>手续费跟白名单未实现，具体根据实际情况添加就好，具体释放思路已在代码注释中！

### 重点讲讲 calculateRelease 计算方法

先计算出有多少个释放时间单位
```solidity
uint256 elapsedTime = block.timestamp - datas[_address].lastUpdateTime;
uint256 elapsedHours = elapsedTime / releaseInterval;
 ```

计算当前地址待释放还有多少币
```solidity
//锁定量 = balance（实际数量）- releaseAmount （已释放数量）
uint256 waitRelease = balanceOf(_address) - datas[_address].releaseAmount;
```

计算每小时应该释放多少币
```solidity
// 每小时应释放多少 = 锁定量 * 0.006 / 24
uint256 releasePerInterval = (waitRelease * 6) / 10000 / 24;
```

根据当前地址释放的时间单位和每小时释放多少币计算出当前时间应该总共释放多少币
```solidity
uint256 releaseAmount = releasePerInterval * elapsedHours;
```


问题：此时会产生一定的时间差，在1小时内，若需要，需要在当前时间减去相差的时间差。 比如：此时过了1:50分，但是计算的只有1小时，那50分钟是丢失的，此时将时间更新为当前时间，是不合理的。在实际项目中一定要解决此问题！

