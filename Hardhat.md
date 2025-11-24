# Hardhat 项目结构和使用指南

本文档介绍 Hardhat 项目的目录结构、各个目录的作用，以及常见的使用方法。这个项目是一个适配 Polkadot 网络的 Uniswap V2 实现，使用 `@parity/hardhat-polkadot` 插件。

## 项目目录结构

### 1. contracts/
存放 Solidity 智能合约文件。

#### 子目录：
- `interfaces/`: 合约接口定义
- `libraries/`: 工具库合约
- `test/`: 测试用合约（通常是用于测试的 ERC20 代币等）

#### 代码示例：
```solidity
// contracts/UniswapV2ERC20.sol (简化版)
pragma solidity =0.8.28;

contract UniswapV2ERC20 {
    string public constant name = 'Uniswap V2';
    string public constant symbol = 'UNI-V2';
    uint8 public constant decimals = 18;

    uint public totalSupply;
    mapping(address => uint) public balanceOf;
    mapping(address => mapping(address => uint)) public allowance;

    event Transfer(address indexed from, address indexed to, uint value);
    event Approval(address indexed owner, address indexed spender, uint value);

    function transfer(address to, uint value) external returns (bool) {
        balanceOf[msg.sender] -= value;
        balanceOf[to] += value;
        emit Transfer(msg.sender, to, value);
        return true;
    }

    function approve(address spender, uint value) external returns (bool) {
        allowance[msg.sender][spender] = value;
        emit Approval(msg.sender, spender, value);
        return true;
    }
}
```

### 2. scripts/
存放部署和交互脚本。

#### 代码示例：
```javascript
// scripts/deploy.js (简化版)
const { ethers } = require("hardhat");

async function main() {
  // 获取签名者账户
  const [deployer] = await ethers.getSigners();
  console.log("部署账户:", deployer.address);

  // 部署工厂合约
  const Factory = await ethers.getContractFactory("UniswapV2Factory");
  const factory = await Factory.deploy(deployer.address);
  await factory.waitForDeployment();

  console.log("工厂合约部署地址:", await factory.getAddress());
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

### 3. test/
存放测试文件，使用 Mocha 框架。

#### 代码示例：
```javascript
// test/UniswapV2Factory.js (简化版)
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("UniswapV2Factory", function () {
  let factory;
  let owner;

  beforeEach(async function () {
    [owner] = await ethers.getSigners();

    const Factory = await ethers.getContractFactory("UniswapV2Factory");
    factory = await Factory.deploy(owner.address);
    await factory.waitForDeployment();
  });

  it("应该正确设置费用接收者", async function () {
    expect(await factory.feeToSetter()).to.equal(owner.address);
  });

  it("应该能创建交易对", async function () {
    // 部署两个测试代币
    const TokenA = await ethers.getContractFactory("ERC20");
    const tokenA = await TokenA.deploy("Token A", "TKA");
    const TokenB = await ethers.getContractFactory("ERC20");
    const tokenB = await TokenB.deploy("Token B", "TKB");

    await factory.createPair(await tokenA.getAddress(), await tokenB.getAddress());
    const pairAddress = await factory.getPair(await tokenA.getAddress(), await tokenB.getAddress());

    expect(pairAddress).to.not.equal(ethers.ZeroAddress);
  });
});
```

#### 测试工具函数：
```javascript
// test/shared/utilities.js (简化版)
const { ethers } = require("ethers");

function expandTo18Decimals(n) {
  return ethers.parseEther(n.toString());
}

async function getWallets(n) {
  const wallets = await ethers.getSigners();
  return wallets.slice(0, n);
}

module.exports = {
  expandTo18Decimals,
  getWallets
};
```

### 4. tasks/
存放自定义 Hardhat 任务。

#### 代码示例：
```javascript
// tasks/custom-task.js
const { task } = require("hardhat/config");

task("custom-task", "执行自定义任务")
  .addParam("param", "任务参数")
  .setAction(async (taskArgs, hre) => {
    console.log("执行自定义任务，参数:", taskArgs.param);

    // 在这里编写任务逻辑
    const accounts = await hre.ethers.getSigners();
    console.log("第一个账户:", accounts[0].address);
  });
```

### 5. 配置文件

#### hardhat.config.js
```javascript
require("@nomicfoundation/hardhat-toolbox");
require("@parity/hardhat-polkadot");
require("dotenv").config();

module.exports = {
  solidity: "0.8.28",
  networks: {
    hardhat: {
      // 本地开发网络配置
    },
    localNode: {
      // 本地 Polkadot 节点配置
      polkavm: true,
      polkadotUrl: "http://127.0.0.1:9944",
      url: "http://127.0.0.1:8545",
      accounts: [process.env.PRIVATE_KEY]
    },
    passetHub: {
      // 测试网配置
      url: "https://testnet-passet-hub-eth-rpc.polkadot.io",
      accounts: [process.env.PRIVATE_KEY]
    }
  },
  mocha: {
    timeout: 100000000
  }
};
```

#### package.json 脚本配置
```json
{
  "scripts": {
    "compile": "hardhat compile",
    "test": "hardhat test",
    "deploy": "hardhat run scripts/deploy.js",
    "node": "hardhat node"
  }
}
```

## 常见 Hardhat 指令使用方法

### 1. 编译合约
```bash
# 编译所有合约
npx hardhat compile

# 或使用 npm 脚本（如果配置了）
npm run compile
```

### 2. 运行测试
```bash
# 运行所有测试
npx hardhat test

# 运行特定测试文件
npx hardhat test test/UniswapV2Factory.js

# 运行包含特定关键词的测试
npx hardhat test --grep "创建交易对"
```

### 3. 部署合约
```bash
# 在默认网络（hardhat）上部署
npx hardhat run scripts/deploy.js

# 在指定网络上部署
npx hardhat run scripts/deploy.js --network localNode
npx hardhat run scripts/deploy.js --network passetHub
```

### 4. 启动本地节点
```bash
# 启动 Hardhat 内置的本地 Ethereum 节点
npx hardhat node

# 启动 Polkadot 本地节点（需要 substrate 节点）
npx hardhat node --polkavm
```

### 5. 控制台交互
```bash
# 启动 Hardhat 控制台
npx hardhat console

# 在指定网络上启动控制台
npx hardhat console --network localNode
```

### 6. 清理编译文件
```bash
# 清理 artifacts 和 cache
npx hardhat clean
```

### 7. 运行自定义任务
```bash
# 运行自定义任务
npx hardhat custom-task --param value
```

### 8. 查看可用任务
```bash
# 显示所有可用任务
npx hardhat
```

### 9. 气体分析
```bash
# 生成气体使用报告
npx hardhat compile && npx hardhat run scripts/gas-analysis.js
```

### 10. 验证合约
```bash
# 在 Etherscan 上验证合约（需要配置 API key）
npx hardhat verify --network mainnet CONTRACT_ADDRESS "Constructor Args"
```

## 环境变量配置 (.env 文件)
```bash
# 私钥配置
LOCAL_PRIV_KEY=your_private_key_here
AH_PRIV_KEY=another_private_key_here

# 网络选择
POLKA_NODE=true  # 是否使用 Polkadot 节点
REVM=true        # 是否使用 REVM（替代 EVM 实现）

# RPC URLs
INFURA_API_KEY=your_infura_api_key
ETHERSCAN_API_KEY=your_etherscan_api_key
```

## 常用开发工作流

1. **开发新功能**：
   ```bash
   # 1. 编写合约代码
   # 2. 编写测试
   npx hardhat test

   # 3. 编译合约
   npx hardhat compile

   # 4. 运行特定测试
   npx hardhat test test/MyContract.js
   ```

2. **部署到测试网**：
   ```bash
   # 1. 配置网络和私钥
   # 2. 编译合约
   npx hardhat compile

   # 3. 部署到测试网
   npx hardhat run scripts/deploy.js --network passetHub
   ```

3. **本地开发和测试**：
   ```bash
   # 1. 启动本地节点
   npx hardhat node

   # 2. 在另一个终端部署合约
   npx hardhat run scripts/deploy.js --network localhost
   ```

这个指南涵盖了 Hardhat 项目的核心结构和使用方法。根据具体需求可以进一步扩展配置和脚本。
