---
title: How to develop ERC20 token faucet
catalog: true
date: 2023-08-12 12:20:17
subtitle: 如何开发一个ERC20代币水龙头
header-img: /img/header_img/lml_bg.jpg
tags:
- Web3-Develop
categories:
- Web3-Develop
---

## Overview | 业务场景：
    一个项目在测试阶段要经过市场和团队内部的多频次测试，要用到代币，为了不是每次都是转账给一个新的账户
    通过实现代币水龙头的功能来快捷的实现领取测试token。

## 具体判断的代码如下：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/security/Pausable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract XERC20 is ERC20, ERC20Burnable, Pausable, Ownable {

    uint256 public amountAllowed = 100;   // Number of claims per time
    mapping(address => bool) requestAddress;  // received

    event MintFaucetToken(address _receive,uint256 _amount);

    constructor() ERC20("XMax", "XMX") {}

    function pause() public onlyOwner {
        _pause();
    }

    function unpause() public onlyOwner {
        _unpause();
    }

    function mint(address to, uint256 amount) public  {
        _mint(to, amount);
    }

    function _beforeTokenTransfer(address from, address to, uint256 amount)
    internal
    whenNotPaused
    override
    {
        super._beforeTokenTransfer(from, to, amount);
    }
    
    //水龙头功能  ，每次请求直接去Mint就好，如果需要添加世间和地址限制，根据需求在此处添加就好
    function requestTokens() external  {
        mint(msg.sender,amountAllowed * 10**18);
        emit MintFaucetToken(msg.sender,amountAllowed * 10**18);

    }
}
```
##### 通过以上，就可以简单实现ERC20代币水龙头功能，仅在测试网用！！！