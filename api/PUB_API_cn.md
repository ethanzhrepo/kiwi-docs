# 公共API接口概述

这些接口不需要签名验证，但有基于IP地址的访问限制（每秒20个请求，峰值允许100个）。

## 通用响应结构
所有API接口返回统一的响应结构：

```json
{
  "code": "整数",
  "msg": "字符串",
  "data": "对象",
  "systemTime": "长整数，系统时间戳"
}
```
响应字段：
- code: 响应状态码
  - 1: 成功
  - 0: 一般错误
- msg: 响应消息
- data: 响应数据对象
- systemTime: 服务器时间戳（毫秒）

## 状态码说明
除了响应结构中的`code`字段外，API还使用HTTP状态码表示请求的状态：
- 200: 成功
- 400: 请求参数错误
- 404: 资源不存在
- 429: 请求频率超过限制
- 500: 服务器内部错误

## 公共API接口

### 1. 获取支持的区块链列表

接口: GET /pub/api/v1/chains

描述: 获取系统支持的所有区块链配置信息（不包括RPC端点）

参数: 无

返回字段:
- name: 链名称
- chainId: 链ID
- symbol: 代币符号
- chainName: 链全名
- decimals: 小数位数
- confirmBlock: 确认区块数
- confirmTimeDelaySeconds: 确认时间延迟（秒）
- confirmThresholdAmountInDecimals: 确认阈值金额
- explorerUrl: 区块浏览器URL
- usdtContracts: USDT合约列表

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": [
    {
      "name": "ETH",
      "chainId": 1,
      "symbol": "ETH",
      "chainName": "Ethereum Mainnet",
      "decimals": 18,
      "confirmBlock": 12,
      "confirmTimeDelaySeconds": 180,
      "confirmThresholdAmountInDecimals": 500000000000000000,
      "explorerUrl": "https://etherscan.io",
      "usdtContracts": [
        {
          "address": "0xdAC17F958D2ee523a2206206994597C13D831ec7",
          "symbol": "USDT",
          "decimals": 6
        }
      ]
    },
    {
      "name": "BSC",
      "chainId": 56,
      "symbol": "BNB",
      "chainName": "Binance Smart Chain",
      "decimals": 18,
      "confirmBlock": 20,
      "confirmTimeDelaySeconds": 60,
      "confirmThresholdAmountInDecimals": 100000000000000000,
      "explorerUrl": "https://bscscan.com",
      "usdtContracts": [
        {
          "address": "0x55d398326f99059fF775485246999027B3197955",
          "symbol": "USDT",
          "decimals": 18
        }
      ]
    }
  ],
  "systemTime": 1678234567890
}
```

### 2. 获取订阅合约地址

接口: GET /pub/api/v1/subscribe/contract

描述: 获取订阅支付使用的智能合约地址

参数: 无

返回示例:
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "contract_address": "0x742d35Cc6634C0532925a3b8D9a77e8B58319f06"
  },
  "systemTime": 1678234567890
}
```

### 3. 报告交易

以下是无需认证的公共接口：

接口: POST /pub/api/v1/report

描述: 报告一个交易哈希以便系统进行处理

请求体:
```json
{
  "txHash": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
  "chainId": 1
}
```

参数:
- txHash: 交易哈希，必须符合格式：0x后跟64个十六进制字符
- chainId: 区块链ID，必须是系统支持的链ID

验证要求:
- 交易哈希必须符合标准格式（0x开头加64个十六进制字符）
- 同一个IP地址10秒内只能提交一次交易哈希
- 同一个交易哈希1分钟内只能被提交一次

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "message": "Transaction processed successfully",
    "txHash": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
    "status": "processed"
  },
  "systemTime": 1678234567890
}
```

同样的接口也提供了需要JWT认证的版本:

接口: POST /pub/api/v1/user/report

描述: 报告一个交易哈希以便系统进行处理，需要JWT认证

请求头:
- Authorization: Bearer {token}

请求体:
```json
{
  "txHash": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
  "chainId": 1
}
```

参数:
- txHash: 交易哈希，必须符合格式：0x后跟64个十六进制字符
- chainId: 区块链ID，必须是系统支持的链ID

验证要求:
- 交易哈希必须符合标准格式（0x开头加64个十六进制字符）
- 同一个用户（mchID + userID组合）在1分钟内只能提交一次交易哈希
- 同一个IP地址10秒内只能提交一次交易哈希
- 同一个交易哈希1分钟内只能被提交一次

返回与公共接口相同。


## 需要JWT认证的用户API接口

以下API接口需要JWT认证，使用Bearer令牌方式进行认证。

### JWT认证要求

对所有`/pub/api/v1/user`下的接口请求，需要在请求头中添加：

- Authorization: Bearer {token}

其中token是使用以下规则生成的JWT令牌：
- 发行者(iss): 商户ID(mchId)
- 受众(aud): 固定值"kiwi"
- 过期时间(exp): 6小时后的时间戳
- 自定义声明:
  - userId: 用户ID

JWT使用商户的secretKey作为密钥，采用HS256算法签名。系统将在每个受保护的接口中，验证请求的用户ID和商户ID与资源所有者是否匹配。

### 1. 获取支付订单详情（需要JWT认证）

接口: GET /pub/api/v1/user/payment/:id

描述: 根据支付ID获取支付订单的详细信息，需要JWT认证

参数:
- id: 支付订单ID (路径变量)

请求头:
- Authorization: Bearer {token}

安全限制:
- 只能访问JWT中指定商户ID和用户ID拥有的支付订单

返回字段:
- id: 支付订单ID (系统生成)
- user_id: 用户ID
- order_id: 商户订单号
- total_fee: 订单总金额
- tax_fee: 税费金额 (已包含在total_fee中)
- expire_at: 订单过期时间
- status: 支付状态 (如 PENDING_PAY, PAID, EXPIRED 等)
- memo: 订单备注 (可选)
- closed_at: 订单关闭时间 (可选, 如果订单已关闭)
- paid_at: 订单支付时间 (可选, 如果订单已支付)
- redirect_url: 支付成功后的跳转URL (可选)
- logo: 订单页显示的Logo URL (可选)

返回示例:
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "P202401011234567890",
    "user_id": "user123",
    "order_id": "merchant_order_123",
    "total_fee": "100.00",
    "tax_fee": "10.00",
    "expire_at": "2023-01-01T12:00:00Z",
    "status": "PENDING_PAY",
    "memo": "测试订单",
    "redirect_url": "https://example.com/success",
    "logo": "https://example.com/logo.png"
  },
  "systemTime": 1678234567890
}
```

### 2. 获取支付/订阅地址（需要JWT认证）

接口: GET /pub/api/v1/user/address/:id

描述: 根据支付ID或订阅ID获取用户的充值地址，需要JWT认证。返回的地址类型（如EVM）和地址本身。

参数:
- id: 支付订单ID (Pxxxxxxxx) 或订阅ID (Sxxxxxxxx) (路径变量)

请求头:
- Authorization: Bearer {token}

安全限制:
- 只能访问JWT中指定商户ID和用户ID拥有的支付订单或订阅对应的地址。
- 系统会验证内部存储地址的签名，确保其未被篡改。

返回字段:
- type: 地址类型 (例如 "EVM")
- address: 用户的充值钱包地址

返回示例:
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "type": "EVM",
    "address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F"
  },
  "systemTime": 1678234567890
}
```

### 3. 查询用户余额（通过支付ID）

接口: GET /pub/api/v1/user/balance/payment/:id

描述: 根据支付ID查询用户在对应商户的账户余额，需要JWT认证

参数:
- id: 支付订单ID (路径变量)

请求头:
- Authorization: Bearer {token}

安全限制:
- 只能访问JWT中指定商户ID和用户ID拥有的支付订单对应的余额

返回字段:
- balance: 用户账户余额

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "balance": "150.00"
  },
  "systemTime": 1678234567890
}
```

### 4. 查询用户余额（通过订阅ID）

接口: GET /pub/api/v1/user/balance/subscribe/:id

描述: 根据订阅ID查询用户在对应商户的账户余额，需要JWT认证

参数:
- id: 订阅ID (路径变量)

请求头:
- Authorization: Bearer {token}

安全限制:
- 只能访问JWT中指定商户ID和用户ID拥有的订阅对应的余额

返回字段:
- balance: 用户账户余额

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "balance": "150.00"
  },
  "systemTime": 1678234567890
}
```

### 5. 使用余额支付订单 (需要JWT认证)

接口: POST /pub/api/v1/user/balance/payment/:id

描述: 使用经过身份验证用户的账户余额来支付指定的支付订单。

参数:
- id: 支付订单ID (Pxxxxxxxx) (路径变量)

请求头:
- Authorization: Bearer {token}

安全限制:
- 只能操作JWT中指定商户ID和用户ID拥有的支付订单。
- 支付订单必须处于可支付状态 (例如 PENDING_PAY)。
- 用户余额必须足够支付订单金额。

请求体: 无

返回字段:
- status: 操作结果 (例如 "success")
- message: 操作结果描述

成功返回示例:
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "status": "success",
    "message": "Payment processed successfully"
  },
  "systemTime": 1678234567890
}
```

失败返回示例 (例如余额不足):
```json
{
  "code": 0,
  "msg": "Insufficient balance to cover the payment",
  "data": null,
  "systemTime": 1678234567890
}
```

失败返回示例 (例如订单状态错误):
```json
{
  "code": 0,
  "msg": "Payment cannot be processed in its current state",
  "data": null,
  "systemTime": 1678234567890
}
```

### 6. 获取订阅详情（需要JWT认证）

接口: GET /pub/api/v1/user/subscribe/:id

描述: 根据订阅ID获取订阅的详细信息，需要JWT认证

参数:
- id: 订阅ID (路径变量)

请求头:
- Authorization: Bearer {token}

安全限制:
- 只能访问JWT中指定商户ID和用户ID拥有的订阅详情

返回字段包含订阅的所有详细信息

### 7. 授权订阅支付

接口: POST /pub/api/v1/user/personalSign/:id

描述: 为订阅提供个人签名授权，需要JWT认证

参数:
- id: 订阅ID (路径变量)

请求头:
- Authorization: Bearer {token}

安全限制:
- 只能为JWT中指定商户ID和用户ID拥有的订阅提供授权

请求体:
```json
{
  "data": "My address:$0xABCDEF... and I want to subscribe to the subscription:$S123456789... at 1678234567",
  "personalSign": "0x签名内容..."
}
```

返回字段:
- status: 成功状态

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "status": "success"
  },
  "systemTime": 1678234567890
}
```

### 8. 确认订阅（无需钱包绑定）

接口: POST /pub/api/v1/user/subscribe/confirm/:id

描述: 确认订阅（无需钱包绑定），需要JWT认证

参数:
- id: 订阅ID (路径变量)

请求头:
- Authorization: Bearer {token}

安全限制:
- 只能确认JWT中指定商户ID和用户ID拥有的订阅
- 订阅状态必须为 PENDING_SUBSCRIBE

返回字段:
- status: 成功状态

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "status": "success"
  },
  "systemTime": 1678234567890
}
```

### 9. 取消订阅

接口: POST /pub/api/v1/user/subscribe/cancel/:id

描述: 取消订阅，需要JWT认证

参数:
- id: 订阅ID (路径变量)

请求头:
- Authorization: Bearer {token}

安全限制:
- 只能取消JWT中指定商户ID和用户ID拥有的订阅
- 订阅状态不能已经是取消状态

返回字段:
- status: 成功状态

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "status": "success"
  },
  "systemTime": 1678234567890
}
```

### 10. 查询订阅账单列表

接口: POST /pub/api/v1/user/subscribe/bills/:id

描述: 查询指定订阅的账单列表，按照创建时间倒序排列，允许按照状态过滤，需要JWT认证

参数:
- id: 订阅ID (路径变量)

请求头:
- Authorization: Bearer {token}

请求体:
```json
{
  "status": "PAID",  // 可选，按账单状态过滤。可选值: PENDING, ON_CHAIN_PROCESSING, PAID, FAILED, CANCELED
  "page": 1,         // 可选，分页页码，默认为1
  "size": 10         // 可选，每页显示数量，默认为10，最大为20
}
```

安全限制:
- 只能查询JWT中指定商户ID和用户ID拥有的订阅的账单
- 订阅必须存在

返回字段:
- bills: 包含订阅账单的详细信息数组
- pagination: 分页信息
  - page: 当前页码
  - size: 每页大小
  - total: 总记录数

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "bills": [
      {
        "id": "B123456789012345678",
        "subscribe_id": "S123456789012345678",
        "mch_id": "mch123456",
        "user_id": "user123456",
        "fee": "10.00",
        "status": "PAID",
        "processed_at": "2023-05-01T12:30:45Z",
        "created_at": "2023-05-01T12:00:00Z",
        "updated_at": "2023-05-01T12:30:45Z",
        "due_date": "2023-05-15T00:00:00Z",
        "bill_date": "2023-05-01T00:00:00Z",
        "billing_period": 1,
        "chain_id": 1,
        "tx_hash": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef"
      },
      {
        "id": "B123456789012345679",
        "subscribe_id": "S123456789012345678",
        "mch_id": "mch123456",
        "user_id": "user123456",
        "fee": "10.00",
        "status": "PENDING",
        "created_at": "2023-04-01T12:00:00Z",
        "due_date": "2023-04-15T00:00:00Z",
        "bill_date": "2023-04-01T00:00:00Z",
        "billing_period": 2
      }
    ],
    "pagination": {
      "page": 1,
      "size": 10,
      "total": 45
    }
  },
  "systemTime": 1678234567890
}
```

### 11. 获取待确认充值记录

接口: GET /pub/api/v1/user/deposits/pending

描述: 获取当前用户的待确认充值交易记录，需要JWT认证。这些交易是已经充值但尚未达到所需确认数的交易。

请求头:
- Authorization: Bearer {token}

安全限制:
- 只能查询JWT中指定商户ID和用户ID的待确认充值记录

返回字段:
- deposits: 包含待确认充值记录的数组，每条记录包含以下字段：
  - id: 交易ID
  - payment_or_subscribe_id: 关联的支付或订阅ID
  - mch_id: 商户ID
  - user_id: 用户ID
  - amount: 金额
  - trade_type: 交易类型（"DEPOSIT"）
  - confirmed_blocks: 已确认区块数
  - required_confirmations: 所需确认区块数
  - need_confirm: 是否需要确认
  - tx_hash: 交易哈希（如有）
  - token: 代币地址（如有）
  - chain_id: 链ID（如有）
  - tx_from: 交易发送地址（如有）
  - tx_to: 交易接收地址（如有）
  - block_number: 区块高度（如有）
  - created_at: 创建时间
  - status: 状态

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "deposits": [
      {
        "id": "T123456789012345678",
        "payment_or_subscribe_id": "P123456789012345678",
        "mch_id": "mch123456",
        "user_id": "user123456",
        "amount": "100.00",
        "trade_type": "DEPOSIT",
        "confirmed_blocks": 3,
        "required_confirmations": 12,
        "need_confirm": true,
        "tx_hash": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
        "token": "0xdAC17F958D2ee523a2206206994597C13D831ec7",
        "chain_id": 1,
        "tx_from": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
        "tx_to": "0x71C7656EC7ab88b098defB751B7401B5f6d8976G",
        "block_number": "15249120",
        "created_at": "2023-05-01T12:00:00Z",
        "status": 1
      }
    ]
  },
  "systemTime": 1678234567890
}
```

## 错误示例

### 认证失败

```json
{
  "code": 0,
  "msg": "Invalid token",
  "data": null,
  "systemTime": 1678234567890
}
```

### 缺少必要的请求头

```json
{
  "code": 0,
  "msg": "Missing or invalid Authorization header",
  "data": null,
  "systemTime": 1678234567890
}
```

### 权限不足

```json
{
  "code": 0,
  "msg": "Payment does not belong to this user",
  "data": null,
  "systemTime": 1678234567890
}
```

### 请求频率超限

```json
{
  "code": 0,
  "msg": "Rate limit exceeded",
  "data": null,
  "systemTime": 1678234567890
}
```

### 资源不存在

```json
{
  "code": 0,
  "msg": "Payment not found",
  "data": null,
  "systemTime": 1678234567890
}
```

### 地址完整性检查失败

```json
{
  "code": 0,
  "msg": "Address integrity check failed",
  "data": null,
  "systemTime": 1678234567890
}
```

### 交易哈希格式无效

```json
{
  "code": 0,
  "msg": "Invalid transaction hash format",
  "data": null,
  "systemTime": 1678234567890
}
```

### 无效的个人签名

```json
{
  "code": 0,
  "msg": "Invalid personal signature",
  "data": null,
  "systemTime": 1678234567890
}
```

### 订单类型错误

```json
{
  "code": 0,
  "msg": "Payment is not a subscription order",
  "data": null,
  "systemTime": 1678234567890
}
```
