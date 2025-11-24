# Uniswap V2 - Polkadot 项目详解

## 项目概述

这个项目是将著名的 Uniswap V2 去中心化交易所协议适配到 Polkadot 生态系统中。Uniswap V2 是以太坊上最广泛使用的 AMM（自动做市商）DEX 协议之一，通过将其实现移植到 Polkadot 生态系统，该项目使用户能够在 Polkadot 上享受类似的去中心化交易体验。

## 核心组件

### 1. UniswapV2Factory.sol - 工厂合约

工厂合约是整个系统的核心，负责创建交易对合约。

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "./interfaces/IUniswapV2Factory.sol";
import "./UniswapV2Pair.sol";

contract UniswapV2Factory is IUniswapV2Factory {
    address public feeTo;
    address public feeToSetter;

    mapping(address => mapping(address => address)) public getPair;
    address[] public allPairs;

    event PairCreated(address indexed token0, address indexed token1, address pair, uint);

    constructor(address _feeToSetter) {
        feeToSetter = _feeToSetter;
    }

    function allPairsLength() external view returns (uint) {
        return allPairs.length;
    }

    function createPair(
        address tokenA,
        address tokenB
    ) external returns (address pair) {
        require(tokenA != tokenB, "UniswapV2: IDENTICAL_ADDRESSES");
        (address token0, address token1) = tokenA < tokenB
            ? (tokenA, tokenB)
            : (tokenB, tokenA);
        require(token0 != address(0), "UniswapV2: ZERO_ADDRESS");
        require(
            getPair[token0][token1] == address(0),
            "UniswapV2: PAIR_EXISTS"
        ); // single check is sufficient
        bytes memory bytecode = type(UniswapV2Pair).creationCode;
        bytes32 salt = keccak256(abi.encodePacked(token0, token1));
        assembly {
            pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
        }
        IUniswapV2Pair(pair).initialize(token0, token1);
        getPair[token0][token1] = pair;
        getPair[token1][token0] = pair; // populate mapping in the reverse direction
        allPairs.push(pair);
        emit PairCreated(token0, token1, pair, allPairs.length);
    }

    function setFeeTo(address _feeTo) external {
        require(msg.sender == feeToSetter, "UniswapV2: FORBIDDEN");
        feeTo = _feeTo;
    }

    function setFeeToSetter(address _feeToSetter) external {
        require(msg.sender == feeToSetter, "UniswapV2: FORBIDDEN");
        feeToSetter = _feeToSetter;
    }
}
```

**关键功能**:
1. `createPair` - 使用 CREATE2 操作码创建新的交易对合约，保证地址可预测
2. 权限管理 - 通过 `feeToSetter` 控制谁可以设置手续费接收地址
3. 交易对追踪 - 维护 `getPair` 映射和 `allPairs` 数组来跟踪所有创建的交易对

### 2. UniswapV2Pair.sol - 交易对合约

交易对合约实现了资金池的所有功能，包括流动性管理和交易执行。

**关键功能**:
1. 流动性管理 - `mint`（添加流动性）和 `burn`（移除流动性）函数
2. 交易执行 - `swap` 函数实现代币交换
3. 价格预言机 - 通过 `price0CumulativeLast` 和 `price1CumulativeLast` 累积价格信息
4. 手续费机制 - `_mintFee` 函数为流动性提供者收取交易费

### 3. UniswapV2ERC20.sol - LP代币合约

这是流动性提供者代币的标准实现，继承自标准 ERC20 并增加了 permit 功能（允许代币授权无需发送交易）。

## 项目运行流程

### 1. 环境设置

项目支持多种运行环境：
1. 标准 EVM 环境（本地测试）
2. PolkaVM 环境（Polkadot 上的 EVM 兼容层）
3. REVM 环境（另一种 EVM 实现）

### 2. 构建过程

1. 安装依赖:
   ```bash
   pnpm install
   ```

2. 构建 Polkadot 二进制文件:
   ```bash
   git clone https://github.com/paritytech/polkadot-sdk
   cd polkadot-sdk
   cargo build --bin substrate-node --release
   cargo build -p pallet-revive-eth-rpc --bin eth-rpc --release
   ```

3. 复制二进制文件到项目 bin 目录:
   ```bash
   mkdir -p ../uniswap-v2-polkadot/bin
   cp ./target/release/substrate-node ../uniswap-v2-polkadot/bin/
   cp ./target/release/eth-rpc ../uniswap-v2-polkadot/bin/
   ```

### 3. 测试流程

1. **标准EVM测试**（推荐用于代码验证）:
   ```bash
   npx hardhat test
   ```

2. **PolkaVM测试**（需要正确配置本地节点）:
   ```bash
   POLKA_NODE=true npx hardhat test --network localNode
   ```

3. **REVM测试**（需要正确配置环境）:
   ```bash
   POLKA_NODE=true REVM=true npx hardhat test --network pvmevm
   ```

### 4. 部署流程

```bash
npx hardhat run scripts/deploy.js --network localNode
```

## 关键技术细节

### 1. CREATE2 地址确定性

在 `UniswapV2Factory.sol` 中，使用 CREATE2 创建交易对合约：

```solidity
bytes memory bytecode = type(UniswapV2Pair).creationCode;
bytes32 salt = keccak256(abi.encodePacked(token0, token1));
assembly {
    pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
}
```

这确保了对于任意两个代币，交易对地址都是确定且可预测的，这对于前端和其他合约非常重要。

### 2. 恒定乘积公式

在 `UniswapV2Pair.sol` 的 `swap` 函数中实现核心的恒定乘积公式 x*y=k：

```solidity
require(
    balance0Adjusted.mul(balance1Adjusted) >=
        uint(_reserve0).mul(_reserve1).mul(1000 ** 2),
    "UniswapV2: K"
);
```

其中 `balance0Adjusted` 和 `balance1Adjusted` 是扣除0.3%交易费后的余额。

### 3. 重入保护

为了防止重入攻击，合约使用了一个简单的锁机制：

```solidity
uint private unlocked = 1;
modifier lock() {
    require(unlocked == 1, "UniswapV2: LOCKED");
    unlocked = 0;
    _;
    unlocked = 1;
}
```

所有关键外部函数（`mint`、`burn`、`swap`）都使用这个修饰符。

### 4. 价格预言机

通过累积价格实现简单的链上价格预言机：

```solidity
price0CumulativeLast +=
    uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) *
    timeElapsed;
price1CumulativeLast +=
    uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) *
    timeElapsed;
```

这允许外部合约计算给定时间窗口内的平均价格。

## 我们所做的改进

在我们的工作中，我们解决了几个关键问题：

1. **账户处理问题**：添加了对账户数量的检查，当只有一个账户时跳过多账户测试
2. **私钥配置问题**：确保所有网络配置都有有效的私钥
3. **网络配置问题**：添加了缺失的 `localNode` 网络配置
4. **测试断言问题**：修改了测试以适应不同环境中的地址变化

## 总结

这个项目成功地将 Uniswap V2 适配到 Polkadot 生态系统，同时保持了原始合约的核心功能。通过我们的修复，代码现在可以在标准 EVM 环境中完全正常运行，并为在 PolkaVM 等特殊环境中运行提供了基础。这为在 Polkadot 生态系统中部署去中心化交易所奠定了坚实的基础。