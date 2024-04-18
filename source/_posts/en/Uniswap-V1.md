---
title: Uniswap V1 深度解析并兼容solidity 0.8版本
catalog: true
date: 2022-08-21 14:39:17
subtitle: uniswap 是一个去中心化交易所协议，而乘积做市是一种流动性提供策略，用于在去中心化交易所中为资产配对提供流动性。
header-img: /img/header_img/lml_bg.jpg
tags:
- Uniswap V1
categories:
- Solidity
---

>Uniswap 是一个去中心化交易所协议，而乘积做市是一种流动性提供策略，用于在去中心化交易所中为资产配对提供流动性。
> 
>- 乘积做市是一种常见的做市策略，它与传统的常数做市（Constant Product Market Making）策略（Uniswap 最早采用的策略）不同。在乘积做市中，做市商会通过为交易对提供资产，使得它们的乘积保持不变。这意味着，当一个交易发生时，交易对中的两种资产数量的乘积保持恒定，从而形成了一个超越线性关系的交易价格曲线。
> 
>- 这种策略的一个重要特点是，在价格接近某一端的时候，交易价格会呈指数级地变化，这在一定程度上能够提供更好的价格敏感性。然而，乘积做市也存在一些挑战，包括对大额交易的抵抗力不足，以及需要在资产价格剧烈波动时频繁调整做市池。
> 
>- 如果您想要了解更多关于 Uniswap 和乘积做市策略的信息，我建议您查阅 Uniswap 官方文档、论坛以及相关的加密货币和区块链资源，以获得更深入的了解。请注意，加密货币领域的技术和概念可能会不断发展和变化，因此最好参考最新的资料。

## 快速开始
> Uniswap V1 源代码 [点这儿](https://github.com/Uniswap/old-solidity-contracts)
> 
> 本文中将用Solidity 0.8版本进行版本兼容
> 
> 开发环境 IDEA， foundry，Hardhat，Solidity ^0.8

### 创建一个新项目并forge初始化

```shell
mkdir UniswapV1
cd UniswapV1
forge init
```

#### forge 安装测试openzeppelin 和forge-std 库

```shell
forge install @openzeppelin-contracts
forge install forge-std 
```

#### 在src目录下创建 interface文件夹
```shell
mkdir  interface
```

#### 编写工厂合约接口
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

interface IUniswapFactory {
    //根据token地址创建交易对
    function launchExchange(address _token) external returns (address exchange);      
    //查询交易对数量
    function getExchangeCount() external view returns (uint exchangeCount);
    //根据token地址查询交易对合约地址
    function tokenToExchangeLookup(address _token) external view returns (address exchange);
    // 根据交易对地址查询token地址
    function exchangeToTokenLookup(address _exchange) external view returns (address token);

}
```

#### 编写交易对合约接口
```solidity
pragma solidity ^0.8.21;
interface IUniSwapExchange{
    //过去交易对ETH交易池数量
    function ethPool() external returns(uint256);
    //获取交易对token交易池数量
    function tokenPool() external returns(uint256);
    // 获取乘积做市的值    K = x * y  此方法获取的是K值
    function invariant() external returns(uint256);
    // 查询总共提供的流动性值
    function totalShares() external returns(uint256);
    // 获取token 代币地址
    function tokenAddress() external returns(address);
    // 获取工厂合约地址
    function factoryAddress() external returns(address);
    // 初始化交易对  根据传入的以太坊数量和token数量计算 乘积做市K值
    function initializeExchange(uint256 _tokenAmount)external payable;
    //用ETH来兑换token
    function ethToTokenSwap(uint256 _minTokens) external payable;
    //用token来兑换ETH
    function tokenToEthSwap(uint256 _tokenAmount, uint256 _minEth ) external;
    //提供流动性
    function investLiquidity(uint256 _minShares) external payable;
    //移除流动性
    function divestLiquidity(uint256 _sharesBurned, uint256 _minEth, uint256 _minTokens) external;
    //获取当前用户提供的流动性
    function getShares(address _provider) external view returns(uint256 _shares);
}
```

#### 编写交易对合约
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;
import {SafeMath} from "@openzeppelin-contracts/contracts/utils/math/SafeMath.sol";
import {IERC20} from "@openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IUniswapFactory} from "./interface/IUniswapFactory.sol";
import {IUniSwapExchange} from "./interface/IUniSwapExchange.sol";


contract UniSwapExchange {
    using SafeMath for uint256;

    event EthToTokenPurchase(address indexed buyer,uint256 indexed ethIn,uint256 indexed tokensOut);  //Eth 購買token事件通知
    event TokenToEthPurchase(address indexed buyer,uint256 indexed tokensIn,uint256 indexed ethOut);  //token 兌換 ETH事件通知
    event Investment(address indexed liquidityProvider, uint256 indexed sharesPurchase);              //添加流動性事件通知
    event Divestment(address indexed liquidityProvider, uint256 indexed sharesBurned);                //移除流動性事件通知
    event InitializeExchange(address indexed token,uint256 indexed initializeAmount);

    uint256 constant FEE_RATE  = 500;            //0.2% 費率

    uint256 public ethPool;
    uint256 public tokenPool;
    uint256 public invariant;
    uint256 public totalShares;
    address public tokenAddress;
    address public factoryAddress;

    mapping(address => uint256) shares;

    IERC20 token;
    IUniswapFactory factory;


    modifier exchangeInitialized(){
        require(invariant > 0 && totalShares > 0);
        _;
    }

    constructor(address _tokenAddress){
        tokenAddress = _tokenAddress;
        factoryAddress = msg.sender;
        token = IERC20(tokenAddress);
        factory = IUniswapFactory(factoryAddress);
    }

    receive() payable external{
        require(msg.value != 0);
        //TODO
        ethToToken(msg.sender, msg.sender, msg.value, 1);
    }

    //初始化交易对
    function initializeExchange(uint256 _tokenAmount)external payable{
        require(invariant == 0 && totalShares == 0);
        require(msg.value >= 10000 && _tokenAmount >= 10000 && msg.value <= 5 * 10 ** 18);

        ethPool = msg.value;
        tokenPool = _tokenAmount;
        invariant = ethPool.mul(tokenPool);
        shares[msg.sender] = 1000;
        totalShares = 1000;
        emit InitializeExchange(tokenAddress,_tokenAmount);
        require(token.transferFrom(payable(msg.sender),address(this),_tokenAmount),"transfer fail!");
    }

    function ethToTokenSwap(uint256 _minTokens) external payable{
        require(msg.value > 0 && _minTokens > 0);
        ethToToken(msg.sender,msg.sender,msg.value,_minTokens);
    }

    function ethToTokenPayment(uint256 _minTokens, address _recipient) external payable {
        require(_minTokens > 0 && _recipient != address(0) && _recipient != address(this));
        ethToToken(msg.sender,_recipient,msg.value,_minTokens);
    }

    function tokenToEthSwap(uint256 _tokenAmount, uint256 _minEth ) external {
        require(_tokenAmount > 0 && _minEth > 0);
        tokenToEth(msg.sender,msg.sender,_tokenAmount,_minEth);
    }

    function tokenToEthPayment(uint256 _tokenAmount ,uint256 _minEth ,address _recipient) external payable{
        require(_tokenAmount > 0 && _minEth > 0 && _recipient != address(0) && _recipient != address(this));
        tokenToEth(msg.sender,_recipient,_tokenAmount,_minEth);
    }

    function tokenToTokenSwap(address _tokenPurchased,address _recipient,uint256 _tokensSold,uint256 _minTokensReceived) external {
        require(_tokenPurchased != address(0) && _recipient != address(0) && _recipient != address(this) && _tokensSold > 0 && _minTokensReceived > 0);
        tokenToTokenOut(_tokenPurchased,msg.sender,_recipient,_tokensSold,_minTokensReceived);
    }

    function tokenTokenIn(address _recipient,uint256 _minTokens) external payable returns(bool){
        require(_minTokens > 0);
        address exchangeToken = factory.exchangeToTokenLookup(msg.sender);
        require(exchangeToken != address(0));
        ethToToken(msg.sender,_recipient, msg.value,_minTokens);
        return true;
    }

    function investLiquidity(uint256 _minShares) external payable exchangeInitialized{
        require(msg.value > 0 && _minShares > 0,"msg.value = 0 && _minShares = 0");
        uint256 ethPerShare = ethPool.div(totalShares);
        require(msg.value >= ethPerShare,"msg.value < ethPerShare");
        uint256 sharesPurchased = msg.value.div(ethPerShare);
        require(sharesPurchased >= _minShares,"sharesPurchased < _minShares");
        uint256 tokensPerShare = tokenPool.div(totalShares);
        uint256 tokensRequired = sharesPurchased.mul(tokensPerShare);
        shares[msg.sender] = shares[msg.sender].add(sharesPurchased);
        totalShares = totalShares.add(sharesPurchased);
        ethPool = ethPool.add(msg.value);
        tokenPool = tokenPool.add(tokensRequired);
        invariant = ethPool.mul(tokenPool);
        emit Investment(msg.sender, sharesPurchased);
        require(token.transferFrom(msg.sender, address(this), tokensRequired),"Test Fail");
    }

    function divestLiquidity(
        uint256 _sharesBurned,
        uint256 _minEth,
        uint256 _minTokens
    )
    external
    {
        require(_sharesBurned > 0);
        shares[msg.sender] = shares[msg.sender].sub(_sharesBurned);
        uint256 ethPerShare = ethPool.div(totalShares);
        uint256 tokensPerShare = tokenPool.div(totalShares);
        uint256 ethDivested = ethPerShare.mul(_sharesBurned);
        uint256 tokensDivested = tokensPerShare.mul(_sharesBurned);
        require(ethDivested >= _minEth && tokensDivested >= _minTokens);
        totalShares = totalShares.sub(_sharesBurned);
        ethPool = ethPool.sub(ethDivested);
        tokenPool = tokenPool.sub(tokensDivested);
        if (totalShares == 0) {
            invariant = 0;
        } else {
            invariant = ethPool.mul(tokenPool);
        }
        emit Divestment(msg.sender, _sharesBurned);
        require(token.transfer(msg.sender, tokensDivested));
        payable(msg.sender).transfer(ethDivested);
    }

    // View share balance of an address
    function getShares(
        address _provider
    )
    external
    view
    returns(uint256 _shares)
    {
        return shares[_provider];
    }

    function ethToToken(address buyer,address recipient,uint256 ethIn,uint256 mintTokensOut) internal exchangeInitialized{

        uint256 fee = ethIn.div(FEE_RATE);                                 //计算费率
        uint256 newEthPool = ethPool.add(ethIn);                           //计算交易对中最新的ETH流动性总量
        uint256 tempEthPool = newEthPool.sub(fee);                         //减去费率后最新流动池中ETH数量
        uint256 newTokenPool = invariant.div(tempEthPool);                 //公式 x * y = k  那最新的tokenPool  y = k / x      10000 = 10 * 1000   10000 / 12 = 833

        uint256 tokensOut = tokenPool.sub(newTokenPool);                   // 为了保持始终 x * y = k  之前的流动池数量 - 重新计算的流动池数量 = 本次买入的token数量

        require(tokensOut >= mintTokensOut && tokensOut <= tokenPool);

        ethPool = newEthPool;                                               // 更新流动性信息
        tokenPool = newTokenPool;
        invariant = newEthPool.mul(newTokenPool);                           //计算新的k值

        emit EthToTokenPurchase(buyer,ethIn,tokensOut);

        require(token.transfer(recipient,tokensOut));
    }


    function tokenToEth(address buyer,address recipient,uint256 tokensIn,uint256 minEthOut) internal exchangeInitialized {

        uint256 fee = tokensIn.div(FEE_RATE);
        uint256 newTokenPool = tokenPool.add(tokensIn);
        uint256 tempTokenPool =  newTokenPool.sub(fee);
        uint256 newEthPool = invariant.div(tempTokenPool);

        uint256 ethOut = ethPool.sub(newEthPool);
        require(ethOut >= minEthOut && ethOut <= ethPool);
        tokenPool = newTokenPool;
        ethPool = newEthPool;
        invariant = newEthPool.mul(newTokenPool);
        emit TokenToEthPurchase(buyer,tokensIn,ethOut);
        require(token.transferFrom(buyer,address(this),tokensIn),"transfer token fail");

        payable(recipient).transfer(ethOut);
    }

    function tokenToTokenOut(address tokenPurchased ,address buyer ,address recipient ,uint256 tokensIn,  uint256 minTokensOut) internal exchangeInitialized{

        require(tokenPurchased != address(0) && tokenPurchased != address(this));
        address exchangeAddress = factory.tokenToExchangeLookup(tokenPurchased);
        require(exchangeAddress != address(0) && exchangeAddress != address(this));

        uint256 fee = tokensIn.div(FEE_RATE);
        uint256 newTokenPool = tokenPool.add(tokensIn);
        uint256 tempTokenPool = newTokenPool.sub(fee);
        uint256 newEthPool = invariant.div(tempTokenPool);
        uint256 ethOut = ethPool.sub(newEthPool);
        require(ethOut <= ethPool);
        IUniSwapExchange exchange = IUniSwapExchange(exchangeAddress);

        emit TokenToEthPurchase(buyer,tokensIn,ethOut);
        tokenPool = newTokenPool;
        ethPool = newEthPool;
        invariant = ethPool.mul(tokenPool);

        require(token.transferFrom(buyer,address(this),tokensIn));
        require(exchange.tokenToTokenIn{value:ethOut}(recipient,minTokensOut));

    }

}
```

#### 编写工厂合约

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.21;
import {UniSwapExchange} from "./UniSwapExchange.sol";

contract UniswapFactory{

    address[] public tokenList;
    mapping(address => address) tokenToExchange;
    mapping(address => address) exchangeToToken;

    event ExchangeLaunch(address indexed exchange, address indexed token);

    function launchExchange(address _token) public returns (address exchange) {
        require(tokenToExchange[_token] == address(0));             //There can only be one exchange per token
        require(_token != address(0) && _token != address(this));
        UniSwapExchange newExchange = new UniSwapExchange(_token);
        tokenList.push(_token);
        tokenToExchange[_token] = address(newExchange);
        exchangeToToken[address(newExchange)] = _token;
        emit ExchangeLaunch(address(newExchange), _token);
        return address(newExchange);
    }


    function getExchangeCount() public view returns (uint exchangeCount) {
        return tokenList.length;
    }

    function tokenToExchangeLookup(address _token) public view returns (address exchange) {
        return tokenToExchange[_token];
    }

    function exchangeToTokenLookup(address _exchange) public view returns (address token) {
        return exchangeToToken[_exchange];
    }
}
```

> 最初版本的UniSwap核心就只有交易对合约和工厂合约，功能也非常简单，就是用以太坊去兑换币，其核心是乘积做市
> 
> - 价格敏感性： 乘积做市策略使得交易价格在接近某一端时呈指数级变化，这意味着在价格变动较小的区间内，交易价格会更为敏感。这可以在市场价格波动较小时提供更好的定价。
>
> - 降低滑点： 由于乘积做市策略的特性，它在价格接近池子的一端时交易价格变化较快。这可以减少交易中的滑点（买卖价格差异），使交易更加准确。
>
> - 激励做市商： 乘积做市可以激励更多的做市商参与，因为他们可以在价格变动较小时也有机会获得利润。这有助于增加交易对的流动性，从而提高交易体验。
>
> - 适应不同市场条件： 乘积做市在不同市场条件下表现良好，无论市场是处于相对稳定还是高度波动的状态。它的价格曲线形状使其能够在多种市场环境下进行有效的价格定价。
>
> - 支持更多资产： 乘积做市策略相对灵活，可以支持不同类型的资产对。这意味着可以用于各种加密货币和数字资产的交易对。
>
> - 去中心化： 乘积做市策略在去中心化交易所中使用，与传统的中心化交易所不同，用户可以在不需要信任中心化机构的情况下进行交易和提供流动性。

### 编译

```solidity
forge build
```
控制台输出：
```text
[⠰] Compiling...
[⠒] Compiling 1 files with 0.8.21
[⠑] Solc 0.8.21 finished in 2.26s
Compiler run successful!
```
编译成功

### Test 测试编写

在foundry Test目录下编写，Test测试文件以 `.t.sol` 结尾

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import {Test, console2} from "forge-std/Test.sol";
import {TestToken} from "../../src/ERC20/ERC20Token.sol";
import {UniswapFactory} from "../../src/UniswapFactory.sol";
import {UniSwapExchange} from "../../src/UniSwapExchange.sol";
import {IUniSwapExchange} from "../../src/interface/IUniSwapExchange.sol";


contract UniswapFactoryTest is Test {
    TestToken public token;
    UniswapFactory public factory;

    //运行Test之前先部署Test token合约和工厂合约
    function setUp() public {
        token = new TestToken();
        factory = new UniswapFactory();
    }

    function test_swap() public {
        //首先创建交易对
        address exchangeAddress = factory.launchExchange(address(token));
        // 加在交易对接口合约
        IUniSwapExchange exchange = IUniSwapExchange(exchangeAddress);
        // 授权token给交易对合约地址 10万token
        token.approve(exchangeAddress,100000 ether);
        // 初始化token  1以太坊和10万token  为初始比例
        exchange.initializeExchange{value: 1 ether}(100000 ether);
        // 使用ETH购买token 传入1 ETH，预期获取49900 代币，实际获取49949.949949949949949950 
        exchange.ethToTokenSwap{value: 1 ether}(49900 ether); //49949.949949949949949950
    }
}
```

uinswap V1 是特别适合新学Solidity的开发者，代码简洁，代码量少，易理解。