# CLI 使用指南

## 概述

`kiwi-cli` 是 Kiwi 支付系统的命令行工具，用于系统管理、商户管理、资金归集等操作。

## 全局配置

### 配置文件位置
- 默认配置文件：`~/.kiwi-cli/config.json`
- EVM 链配置：`~/.kiwi-cli/evm_chains.json`
- 管理员密钥存储：`~/.kiwi-cli/` 目录

### 全局标志
- `--config`: 指定配置文件路径

## 配置管理 (config)

### 基本用法
```bash
kiwi-cli config set [key] [value]   # 设置配置项
kiwi-cli config get [key]           # 获取配置项
kiwi-cli config list                # 列出所有配置项
kiwi-cli config delete [key]        # 删除配置项
```

配置项保存在 `~/.kiwi-cli/config.json` 中。

## 管理员命令 (admin)

### 生成 RSA 密钥对

#### 生成管理员 RSA 密钥
```bash
kiwi-cli admin generate-admin-rsa \
    --save google,dropbox,box,keychain,fs \
    --path [密钥名称] \
    --save-public
```

**参数说明：**
- `--save`: 存储方式（必选），支持：google、dropbox、box、keychain、fs
- `--path`: 密钥保存路径
- `--save-public`: 是否保存公钥到 `~/.kiwi-cli/admin_pool_pub.pem`（默认：true）

#### 生成 CLI 通信密钥
```bash
kiwi-cli admin generate-cli-rsa \
    --force    # 强制覆盖已存在的私钥文件
```

生成 CLI 通信用的 4096 位 RSA 密钥对，私钥使用 AES 加密保存在 `~/.kiwi-cli/admin_cli_private.pem`。

#### RSA 签名
```bash
kiwi-cli admin rsa-sign \
    --provider fs \
    --encrypted-private-key-file [私钥文件路径] \
    --data [待签名数据]
```

使用 RSA-PSS 和 SHA384 对数据进行签名。

## 商户管理 (merchant)

### 商户操作

#### 列出商户
```bash
kiwi-cli merchant list \
    --all          # 列出所有商户（无分页）
    --page 1       # 页码（默认：1）
    --size 10      # 每页大小（默认：10）
```

#### 添加商户
```bash
kiwi-cli merchant add \
    --name [商户名称] \
    --notify-url [通知 URL]
```

**参数说明：**
- `--name`: 商户名称（必选）
- `--notify-url`: 通知回调 URL（可选）

#### 删除商户
```bash
kiwi-cli merchant delete --id [商户ID]
```

#### 重置商户 API 密钥
```bash
kiwi-cli merchant reset-key --id [商户ID]
```

## 钱包管理 (wallet)

### 钱包地址管理

#### 添加钱包地址
```bash
kiwi-cli wallet add \
    --count [生成数量] \
    --type [链类型]
```

批量生成收款钱包地址，钱包私钥使用 AES-256-GCM 加密。

#### 获取钱包信息
```bash
kiwi-cli wallet get [参数]
```

#### 启用/禁用钱包
```bash
kiwi-cli wallet enable [钱包参数]      # 启用钱包
kiwi-cli wallet disable [钱包参数]     # 禁用钱包
kiwi-cli wallet enable-batch [批次]    # 批量启用
kiwi-cli wallet disable-batch [批次]   # 批量禁用
```

#### 钱包统计
```bash
kiwi-cli wallet stat
```

显示钱包池统计信息。

## 资金归集 (collect)

### 归集任务管理

#### 列出归集任务
```bash
kiwi-cli collect collect-task list \
    --status [状态]    # 按状态过滤：pending, collected
```

#### 创建归集任务
```bash
kiwi-cli collect collect-task create \
    --threshold 0      # 归集阈值（低于此金额不归集）
```

#### 获取任务详情
```bash
kiwi-cli collect collect-task get [taskId]
```

#### 删除任务
```bash
kiwi-cli collect collect-task delete [taskId]
```

### 执行归集

```bash
kiwi-cli collect run \
    --task-id [任务ID] \
    --to [归集目标地址] \
    --relayer-provider [中继钱包提供商] \
    --relayer-private-key-file [中继钱包文件] \
    --encrypted-rsa-admin-key-file [加密的 RSA 私钥文件]
```

**必选参数：**
- `--task-id`: 要执行归集的任务 ID
- `--to`: 资金归集的目标地址
- `--relayer-provider`: 中继钱包提供商（google、fs、keychain、dropbox、s3）
- `--relayer-private-key-file`: 中继钱包文件路径
- `--encrypted-rsa-admin-key-file`: 用于解密钱包密钥的加密 RSA 私钥文件

**可选参数：**
- `--auto-7702`: 使用 EIP-7702 协议（默认：true）
- `--batch-size`: 每个交易最大地址数（默认：200）
- `--report-dir`: 归集报告保存目录（默认：./reports）
- `--template-address`: EIP-7702 模板合约地址
- `--token-filter`: 要归集的代币过滤（逗号分隔的代币符号列表）
- `--gas-price`: 自定义 gas 价格
- `--max-gas-cost`: 最大 gas 成本
- `--gas-multiplier`: gas 价格倍数（默认：1.0）
- `--log-level`: 日志级别（默认：info）
- `--log-file`: 日志文件
- `--log-format`: 日志格式（默认：text）
- `--retry-failed`: 重试失败交易的 CSV 文件路径

## 交易管理 (transaction)

### 查询订单
```bash
kiwi-cli transaction get-order [order_id]
```

根据订单 ID 自动检测订单类型：
- P 前缀：支付订单
- S 前缀：订阅订单（包含关联账单）
- B 前缀：订阅账单

### 查询交易
```bash
kiwi-cli transaction get-tx [tx_hash]
```

根据交易哈希获取交易详情。

## 系统管理 (system)

### 系统状态

#### 检查服务器连通性
```bash
kiwi-cli system ping
```

#### 查看服务器状态
```bash
kiwi-cli system server-status
```

显示服务器状态和指标，包括 CPU、内存使用情况和 EVM 链监听器状态。

### Nonce 管理

#### 同步所有 nonce
```bash
kiwi-cli system nonce sync
```

#### 获取 nonce 状态
```bash
kiwi-cli system nonce status <chain_id> <address>
```

#### 同步单个地址 nonce
```bash
kiwi-cli system nonce sync-single <chain_id> <address>
```

#### 验证 nonce 序列
```bash
kiwi-cli system nonce validate
```

## 工具集 (tools)

### 生成 OTP
```bash
kiwi-cli tools generate-otp
```

生成用于 Google Authenticator 的 OTP 密钥，生成后显示在屏幕上并提供配置指导。

### 空投管理
```bash
kiwi-cli tools airdrop [参数]
```

空投手续费管理，用于资金归集前的手续费空投。

### 系统统计
```bash
kiwi-cli tools stat
```

显示系统统计信息，包括充值、支付和商户数据。

## 安全特性

1. **密钥管理**：支持多种存储提供商（文件系统、云存储、HSM）
2. **加密保护**：RSA 加密、HSM 支持和安全密钥管理
3. **批量操作**：许多命令支持批量处理以提高效率
4. **多链支持**：支持多个区块链网络
5. **高级功能**：EIP-7702 支持，用于高效代币归集

## 批量空投合约地址

所有 EVM 链通用：
- 合约地址：`0xE9511e55d2AaC1F62D7e3110f7800845dB2a31F1`
- TRON 链：`TNnHipM7aZMYYanXhESgRV9NmjndcgvaXu`

参考项目：[https://github.com/WhiteRiverBay/wrb-airdrop-contract](https://github.com/WhiteRiverBay/wrb-airdrop-contract)

## 批量查询地址

EVM 多调用合约：
- 合约地址：`0x058C6121efBF3e7C1f856928f7e9ecBC71c5772a`

参考项目：[https://github.com/WhiteRiverBay/evm-multicall](https://github.com/WhiteRiverBay/evm-multicall)

## 使用示例

### 完整的商户设置流程
```bash
# 1. 设置 API 端点
kiwi-cli config set api_url https://api.example.com

# 2. 生成管理员密钥
kiwi-cli admin generate-admin-rsa --save fs --path admin_keys

# 3. 添加商户
kiwi-cli merchant add --name "示例商户" --notify-url https://merchant.com/notify

# 4. 生成钱包地址
kiwi-cli wallet add --count 1000 --type ETH

# 5. 查看系统状态
kiwi-cli system server-status
```

### 资金归集流程
```bash
# 1. 创建归集任务
kiwi-cli collect collect-task create --threshold 0.001

# 2. 查看任务列表
kiwi-cli collect collect-task list

# 3. 执行归集
kiwi-cli collect run \
    --task-id TASK123 \
    --to 0x1234567890abcdef \
    --relayer-provider fs \
    --relayer-private-key-file ./relayer.key \
    --encrypted-rsa-admin-key-file ./admin.pem
```

有关 CLI 交互的管理员 REST API 详细信息，请参考 [CLI 管理员接口概述 (ADMIN_REST_API_cn.md)](docs/ADMIN_REST_API_cn.md)。