---
title: Use foundry forge to build smart contract test case
catalog: true
date: 2023-08-16 23:31:17
subtitle: 使用foundry forge构建智能合约测试用例
header-img: /img/header_img/lml_bg.jpg
tags:
- Web3-Develop
categories:
- Web3-Develop
---

## Quick Start  


## Installation

>预先安装:  [rust](https://www.rust-lang.org/tools/install) 、git
> 

### 安装foundry

>[官方文档](https://book.getfoundry.sh/getting-started/installation#installation) 
> 
> [中文文档](https://coldstar1993.github.io/foundrybook/zh/index.html)


### Windows 安装foundry

```shell
git clone https://github.com/foundry-rs/foundry.git
cd foundry
# install Forge + Cast
cargo install --path ./crates/cli --profile local --bins --force
# install Anvil
cargo install --path ./crates/anvil --profile local --force
# install Chisel
cargo install --path ./crates/chisel --profile local --force
```

### 检查安装是否成功

```shell
$ forge --version

forge 0.2.0 (d154507 2023-08-16T03:30:27.548281700Z)
```
返回版本号，说明已经安装完成

## 创建一个新项目

### 初始化一个新项目

```shell
forge init Test
```

初始化成功后的目录： 
```shell
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         2023/8/16     21:26                .github
d-----         2023/8/16     21:26                lib      
d-----         2023/8/16     21:26                script
d-----         2023/8/16     21:26                src
d-----         2023/8/16     21:26                test
-a----         2023/8/16     21:26            170 .gitignore
-a----         2023/8/16     21:26            148 foundry.toml
-a----         2023/8/16     21:26           1046 README.md
```

### 安装  forge-std

```shell
$ forge install foundry-rs/forge-std
```

因为网络问题，可能会出错，多尝试几次！

> 注意！！！ 请检查 `\lib\forge-std\lib\ds-test` 是否为空，如果为空删除掉此文件夹。
> 
> 删除掉后在\lib\forge-std\lib 目录执行 git clone https://github.com/dapphub/ds-test.git
> 
> 原因：因为网络问题，ds-test不能随forge-std一起下载下来，那就手动下载下来放在该目录下

### 安装`OpenZeppelin/openzeppelin-contracts` 合约库

```shell
$ forge install OpenZeppelin/openzeppelin-contracts
```
因为网络问题，可能会出错，多尝试几次！

> 注意！！！  此时如果出现此错误

```text
Error:
The target directory is a part of or on its own an already initialized git repository,
and it requires clean working and staging areas, including no untracked files.

Check the current git repository's status with `git status`.
Then, you can track files with `git add ...` and then commit them with `git commit`,
ignore them in the `.gitignore` file, or run this command again with the `--no-commit` flag.

If none of the previous steps worked, please open an issue at:
https://github.com/foundry-rs/foundry/issues/new/choose
```

请运行此命令：
```shell
$ forge install OpenZeppelin/openzeppelin-contracts  --no-commit
```

### 编译项目

```shell
$ forge build
```

```text
[⠒] Compiling...
[⠰] Compiling 22 files with 0.8.21
[⠒] Solc 0.8.21 finished in 6.85s
Compiler run successful!
```


### 执行测试

```shell
$ forge  test
```

```text
[PASS] testFuzz_SetNumber(uint256) (runs: 256, μ: 26987, ~: 28387)
[PASS] test_Increment() (gas: 28357)
Test result: ok. 2 passed; 0 failed; 0 skipped; finished in 22.52ms
Ran 1 test suites: 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

### 执行测试并打印出log执行详细日志

> -v :默认不打印  -vv : 一般    -vvv: 日志级别打印   -vvvv:详细打印  -vvvvv :运行详情traces 打印    

```shell
$ forge  test -vvvvv
```

```text
[⠰] Compiling...
No files changed, compilation skipped

Running 2 tests for test/Counter.t.sol:CounterTest
[PASS] testFuzz_SetNumber(uint256) (runs: 256, μ: 27998, ~: 28387)
Traces:
  [106719] CounterTest::setUp()
    ├─ [49499] → new Counter@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   └─ ← 247 bytes of code
    ├─ [2390] Counter::setNumber(0)
    │   └─ ← ()
    └─ ← ()

  [28387] CounterTest::testFuzz_SetNumber(18)
    ├─ [22290] Counter::setNumber(18)
    │   └─ ← ()
    ├─ [283] Counter::number() [staticcall]
    │   └─ ← 18
    └─ ← ()

[PASS] test_Increment() (gas: 28357)
Traces:
  [106719] CounterTest::setUp()
    ├─ [49499] → new Counter@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   └─ ← 247 bytes of code
    ├─ [2390] Counter::setNumber(0)
    │   └─ ← ()
    └─ ← ()

  [28357] CounterTest::test_Increment()
    ├─ [22340] Counter::increment()
    │   └─ ← ()
    ├─ [283] Counter::number() [staticcall]
    │   └─ ← 1
    └─ ← ()

Test result: ok. 2 passed; 0 failed; 0 skipped; finished in 26.24ms
Ran 1 test suites: 2 tests passed, 0 failed, 0 skipped (2 total tests)
```


## 新建一个ERC20代币的项目，并写`transfer` Test case

### 在`src`目录新建`XToken.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import  "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import  "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import  "@openzeppelin/contracts/security/Pausable.sol";
import  "@openzeppelin/contracts/access/Ownable.sol";

contract XToken is ERC20, ERC20Burnable, Pausable, Ownable {
    uint256 public amountAllowed = 10000; // Number of claims per time


    event MintFaucetToken(address _receive, uint256 _amount);

    constructor(string memory name,string memory symbol) ERC20(name, symbol) Ownable(msg.sender) {
        _mint(msg.sender, 100000000 * 10 ** 18 );
    }

    function pause() public onlyOwner {
        _pause();
    }

    function unpause() public onlyOwner {
        _unpause();
    }

    function mint(address to, uint256 amount) public {
        _mint(to, amount);
    }

    //领取测试币
    function requestTokens() external {
        mint(msg.sender, amountAllowed * 10**18);
        emit MintFaucetToken(msg.sender, amountAllowed * 10**18);
    }
}
```

### 编译 
```shell
forge build
```

### 在`test`目录新建`XToken.t.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {XToken} from "../src/XToken.sol";

contract XTokenTest is Test {
    XToken public token;

    address alice = address(1);
    address bob = address(2);

    function setUp() public {
        token = new XToken("XXX","X");
        console.log("XToken address:",address(this));
    }

    function testTransfer() public {
        token.transfer(alice,100 * 10 ** 18);
        console.log("current time:",block.timestamp);
        vm.warp(10 days);  //跳过10天时间
        token.transfer(bob,100 * 10 ** 18);
        console.log("skip 10 days:",block.timestamp);
    }

    function testRequestTokens() public {
        token.requestTokens();
    }
}
```

### 测试

```shell
$ forge test --match-contract XTokenTest -vvvvv
```

```text
[⠆] Compiling...
[⠊] Compiling 1 files with 0.8.21
[⠒] Solc 0.8.21 finished in 1.76s
Compiler run successful!

Running 2 tests for test/XToken.t.sol:XTokenTest
[PASS] testRequestTokens() (gas: 21372)
Logs:
  XToken address: 0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496

Traces:
  [818417] XTokenTest::setUp()
    ├─ [759726] → new XToken@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   ├─ emit OwnershipTransferred(previousOwner: 0x0000000000000000000000000000000000000000, newOwner: XTokenTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: XTokenTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 100000000000000000000000000 [1e26])
    │   └─ ← 3102 bytes of code
    ├─ [0] console::log(XToken address:, XTokenTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496]) [staticcall]
    │   └─ ← ()
    └─ ← ()

  [21372] XTokenTest::testRequestTokens()
    ├─ [16205] XToken::requestTokens()
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: XTokenTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 10000000000000000000000 [1e22])
    │   ├─ emit MintFaucetToken(_receive: XTokenTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], _amount: 10000000000000000000000 [1e22])
    │   └─ ← ()
    └─ ← ()

[PASS] testTransfer() (gas: 71906)
Logs:
  XToken address: 0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496
  current time: 1
  skip 10 days: 864000

Traces:
  [818417] XTokenTest::setUp()
    ├─ [759726] → new XToken@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   ├─ emit OwnershipTransferred(previousOwner: 0x0000000000000000000000000000000000000000, newOwner: XTokenTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: XTokenTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 100000000000000000000000000 [1e26])
    │   └─ ← 3102 bytes of code
    ├─ [0] console::log(XToken address:, XTokenTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496]) [staticcall]
    │   └─ ← ()
    └─ ← ()

  [71906] XTokenTest::testTransfer()
    ├─ [30010] XToken::transfer(0x0000000000000000000000000000000000000001, 100000000000000000000 [1e20])
    │   ├─ emit Transfer(from: XTokenTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], to: 0x0000000000000000000000000000000000000001, value: 100000000000000000000 [1e20])
    │   └─ ← true
    ├─ [0] console::9710a9d0(00000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000000d63757272656e742074696d653a0000000000000000000000000000000
0000000) [staticcall]
    │   └─ ← ()
    ├─ [0] VM::warp(864000 [8.64e5])
    │   └─ ← ()
    ├─ [25210] XToken::transfer(0x0000000000000000000000000000000000000002, 100000000000000000000 [1e20])
    │   ├─ emit Transfer(from: XTokenTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], to: 0x0000000000000000000000000000000000000002, value: 100000000000000000000 [1e20])
    │   └─ ← true
    ├─ [0] console::9710a9d0(000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000d2f00000000000000000000000000000000000000000000000000000000000000000d736b697020313020646179733a0000000000000000000000000000000
0000000) [staticcall]
    │   └─ ← ()
    └─ ← ()

Test result: ok. 2 passed; 0 failed; 0 skipped; finished in 2.06ms
```

至此一个简单的forge  smart contract Test case 项目开发完毕！！！

最后看看`forge`命令
```shell
$ forge -h
```
```text
  bind               Generate Rust bindings for smart contracts
  build              Build the project's smart contracts [aliases: b, compile]
  cache              Manage the Foundry cache
  clean              Remove the build artifacts and cache directories [aliases: cl]
  completions        Generate shell completions script [aliases: com]
  config             Display the current config [aliases: co]
  coverage           Generate coverage reports
  create             Deploy a smart contract [aliases: c]
  debug              Debugs a single smart contract as a script [aliases: d]
  doc                Generate documentation for the project
  flatten            Flatten a source file and all of its imports into one file [aliases: f]
  fmt                Format Solidity source files
  geiger             Detects usage of unsafe cheat codes in a project and its dependencies
  generate           Generate scaffold files
  generate-fig-spec  Generate Fig autocompletion spec [aliases: fig]
  help               Print this message or the help of the given subcommand(s)
  init               Create a new Forge project
  inspect            Get specialized information about a smart contract [aliases: in]
  install            Install one or multiple dependencies [aliases: i]
  remappings         Get the automatically inferred remappings for the project [aliases: re]
  remove             Remove one or multiple dependencies [aliases: rm]
  script             Run a smart contract as a script, building transactions that can be sent onchain
  selectors          Function selector utilities [aliases: se]
  snapshot           Create a snapshot of each test's gas usage [aliases: s]
  test               Run the project's tests [aliases: t]
  tree               Display a tree visualization of the project's dependency graph [aliases: tr]
  update             Update one or multiple dependencies [aliases: u]
  upload-selectors   Uploads abi of given contract to the https://api.openchain.xyz function selector database
                         [aliases: up]
  verify-check       Check verification status on Etherscan [aliases: vc]
  verify-contract    Verify smart contracts on Etherscan [aliases: v]
```

```shell
$ forge test -h
```

```text
Test options:
      --debug <TEST_FUNCTION>    Run a test in the debugger
      --gas-report               Print a gas report [env: FORGE_GAS_REPORT=]
      --allow-failure            Exit with code 0 even if a test fails [env: FORGE_ALLOW_FAILURE=]
      --fail-fast                Stop running tests after the first failure
      --etherscan-api-key <KEY>  The Etherscan (or equivalent) API key [env: ETHERSCAN_API_KEY=]
      --fuzz-seed <FUZZ_SEED>    Set seed used to generate randomness during your fuzz runs
      --fuzz-runs <RUNS>         [env: FOUNDRY_FUZZ_RUNS=]

Display options:
  -j, --json  Output test results in JSON format
  -l, --list  List tests instead of running them

Test filtering:
      --match-test <REGEX>         Only run test functions matching the specified regex pattern [aliases: mt]
      --no-match-test <REGEX>      Only run test functions that do not match the specified regex pattern [aliases: nmt]
      --match-contract <REGEX>     Only run tests in contracts matching the specified regex pattern [aliases: mc]
      --no-match-contract <REGEX>  Only run tests in contracts that do not match the specified regex pattern [aliases:
                                   nmc]
      --match-path <GLOB>          Only run tests in source files matching the specified glob pattern [aliases: mp]
      --no-match-path <GLOB>       Only run tests in source files that do not match the specified glob pattern [aliases:
                                   nmp]

EVM options:
  -f, --fork-url <URL>                Fetch state over a remote endpoint instead of starting from an empty state
                                      [aliases: rpc-url]
      --fork-block-number <BLOCK>     Fetch state from a specific block number over a remote endpoint
      --fork-retry-backoff <BACKOFF>  Initial retry backoff on encountering errors
      --no-storage-caching            Explicitly disables the use of RPC caching
      --initial-balance <BALANCE>     The initial balance of deployed test contracts
      --sender <ADDRESS>              The address which will be executing tests
      --ffi                           Enable the FFI cheatcode
  -v, --verbosity...                  Verbosity of the EVM.

Fork config:
      --compute-units-per-second <CUPS>  Sets the number of assumed available compute units per second for this provider
      --no-rpc-rate-limit                Disables rate limiting for this node's provider [aliases: no-rate-limit]

Executor environment config:
      --gas-limit <GAS_LIMIT>          The block gas limit
      --code-size-limit <CODE_SIZE>    EIP-170: Contract code size limit in bytes. Useful to increase this because of
                                       tests. By default, it is 0x6000 (~25kb)
      --chain-id <CHAIN_ID>            The chain ID
      --gas-price <GAS_PRICE>          The gas price
      --block-base-fee-per-gas <FEE>   The base fee in a block [aliases: base-fee]
      --tx-origin <ADDRESS>            The transaction origin
      --block-coinbase <ADDRESS>       The coinbase of the block
      --block-timestamp <TIMESTAMP>    The timestamp of the block
      --block-number <BLOCK>           The block number
      --block-difficulty <DIFFICULTY>  The block difficulty
      --block-prevrandao <PREVRANDAO>  The block prevrandao value. NOTE: Before merge this field was mix_hash
      --block-gas-limit <GAS_LIMIT>    The block gas limit
      --memory-limit <MEMORY_LIMIT>    The memory limit of the EVM in bytes (32 MB by default)

Cache options:
      --force  Clear the cache and artifacts folder and recompile

Linker options:
      --libraries <LIBRARIES>  Set pre-linked libraries [env: DAPP_LIBRARIES=]

Compiler options:
      --ignored-error-codes <ERROR_CODES>  Ignore solc warnings by error code
      --deny-warnings                      Warnings will trigger a compiler error
      --no-auto-detect                     Do not auto-detect the `solc` version
      --use <SOLC_VERSION>                 Specify the solc version, or a path to a local solc, to build with
      --offline                            Do not access the network
      --via-ir                             Use the Yul intermediate representation compilation pipeline
      --silent                             Don't print anything on startup
      --evm-version <VERSION>              The target EVM version
      --optimize                           Activate the Solidity optimizer
      --optimizer-runs <RUNS>              The number of optimizer runs
      --extra-output <SELECTOR>...         Extra output to include in the contract's artifact
      --extra-output-files <SELECTOR>...   Extra output to write to separate files

Project options:
  -o, --out <PATH>               The path to the contract artifacts folder
      --revert-strings <REVERT>  Revert string configuration
      --build-info               Generate build info files
      --build-info-path <PATH>   Output path to directory that build info files will be written to
      --root <PATH>              The project's root path
  -C, --contracts <PATH>         The contracts source directory
  -R, --remappings <REMAPPINGS>  The project's remappings
      --remappings-env <ENV>     The project's remappings from the environment
      --cache-path <PATH>        The path to the compiler cache
      --lib-paths <PATH>         The path to the library folder
      --hardhat                  Use the Hardhat-style project layout [aliases: hh]
      --config-path <FILE>       Path to the config file

Watch options:
  -w, --watch [<PATH>...]    Watch the given files or directories for changes
      --no-restart           Do not restart the command while it's still running
      --run-all              Explicitly re-run all tests when a change is made
      --watch-delay <DELAY>  File update debounce delay
```