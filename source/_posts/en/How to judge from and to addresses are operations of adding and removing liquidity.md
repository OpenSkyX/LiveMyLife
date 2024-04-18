---
title: Smart contract | How to judge from and to addresses are operations of adding and removing liquidity
catalog: true
date: 2023-08-11 1:20:17
subtitle: 智能合约 | 如何判断from和to地址是添加和删除流动性的操作
header-img: /img/header_img/lml_bg.jpg
tags:
- Web3-Develop
categories:
- Web3-Develop
---

## Overview | 业务场景：
    最近遇到一个项目，需要发行一个ERC20 Token ,除去普通钱包转账，和Swap添加流动性和
    删除流动性之外的transfer都需要收取手续费，这个时候我们需要写一个判断是否为添加流动
    性和删除流动性的判断

## 具体判断的代码如下：

```solidity
contract LiquidityActivity {
    
    //check address is a add liquidity address
    function isAddLiquidity(address from, address to, address liquidityPoolAddress) external view returns (bool) {
        IUniswapV2Pair pool = IUniswapV2Pair(liquidityPoolAddress);

        // Check if `from` is the liquidity pool contract and `to` is a user
        if (from == liquidityPoolAddress && !isContract(to)) {
            // Check if `to` holds both tokens of the pool (add liquidity)
            return (to == pool.token0() || to == pool.token1());
        }

        return false;
    }
    
    //check address is remove liquidity address
    function isRemoveLiquidity(address from, address to, address liquidityPoolAddress) external view returns (bool) {
        IUniswapV2Pair pool = IUniswapV2Pair(liquidityPoolAddress);

        // Check if `to` is the liquidity pool contract and `from` is a user
        if (to == liquidityPoolAddress && !isContract(from)) {
            // Check if `from` holds both tokens of the pool (remove liquidity)
            return (from == pool.token0() || from == pool.token1());
        }

        return false;
    }

    // check address is a contract address
    function isContract(address addr) internal view returns (bool) {
        uint32 size;
        assembly {
            size := extcodesize(addr)
        }
        return (size > 0);
    }
}
```

##### 通过以上判断，就可以分辨出是否为添加和删除流动性操作，进而不对添加和删除流动性的操作收取手续费