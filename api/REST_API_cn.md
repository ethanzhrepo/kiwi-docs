# 应用API 接口概述

**所有的API请求都必须包含标准请求头: X-MCH-ID、X-Timestamp、X-Signature、X-Nonce**

每个独立的商户(mchId)都有独立的 mch_id和secret.

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
响应字段
    http状态码都使用200，使用json中的code码来表示结果
code: 响应状态码
 - 1: 成功
 - 0: 一般错误
msg: 响应消息
data: 响应数据对象
systemTime: 服务器时间戳（毫秒）

## 费用说明

**重要**: 所有接口中的费用(`fee`、`totalFee`等)均已包含税费部分，`taxFee`字段仅作说明用途，不会额外计入总金额。例如，如果`totalFee`为100元，其中`taxFee`为10元，则实际收取用户的总金额为100元，而非110元。

## 通知机制

系统会通过HTTP POST请求向商户配置的通知URL发送事件通知。通知请求包含以下HTTP头：

- **Content-Type**: application/json
- **X-KIWI-SIGN**: 通知签名，用于验证通知的真实性
- **X-KIWI-TIMESTAMP**: 请求时间戳（Unix秒格式）
- **X-KIWI-NONCE**: 随机字符串，用于防止重放攻击

### 通知签名验证

通知签名使用规范化的方法生成，与API请求签名使用相同的基本规则。商户应通过以下步骤验证通知的真实性：

1. 构建规范化字符串：`"POST" + 通知URL + "&" + Base64(通知内容) + "&timestamp=" + timestamp + "&nonce=" + nonce + "&key=" + secretKey`
2. 使用商户密钥计算该字符串的HMAC-SHA256哈希值，并将结果转换为十六进制字符串
3. 将计算出的签名与通知请求中的`X-KIWI-SIGN`头进行比较

示例代码 (Go):
```go
func verifyNotificationSignature(notifyURL string, requestBody []byte, signature, timestamp, nonce, secretKey string) bool {
    // 构建规范化字符串
    timestampInt, _ := strconv.ParseInt(timestamp, 10, 64)
    
    // 需要base64编码请求体
    encodedBody := base64.StdEncoding.EncodeToString(requestBody)
    
    // 构建完整的规范化字符串
    canonicalString := "POST" + notifyURL + "&" + encodedBody + 
                       "&timestamp=" + timestamp + 
                       "&nonce=" + nonce + 
                       "&key=" + secretKey
    
    // 计算HMAC-SHA256哈希
    mac := hmac.New(sha256.New, []byte(secretKey))
    mac.Write([]byte(canonicalString))
    calculatedSignature := hex.EncodeToString(mac.Sum(nil))
    
    // 比较签名
    return calculatedSignature == signature
}
```

### 通知处理要求

商户在处理通知时应注意：
1. 通知可能会重复发送，系统会进行多次重试，请进行幂等性处理
2. 必须返回HTTP状态码200表示接收成功，否则系统会视为通知失败并进行重试
3. 重试间隔依次为：1分钟、5分钟、10分钟、30分钟、1小时等，最长重试24小时
4. 同一个事件只会通知一次，除非系统未收到成功响应

### 支付相关通知

支付相关通知会在支付状态发生变化时发送，包括以下事件类型：

1. **支付成功通知** (event_type: PAYMENT_SUCCESS)
   - 当用户完成支付后，系统确认交易有效
   - 包含支付ID、订单金额、交易哈希等信息

2. **支付失败通知** (event_type: PAYMENT_FAILED)
   - 当支付处理失败时
   - 包含支付ID和失败原因

3. **订单关闭通知** (event_type: PAYMENT_CLOSED)
   - 当订单被手动关闭或自动过期时
   - 包含支付ID和关闭原因

4. **订单退款通知** (event_type: PAYMENT_REFUNDED)
   - 当订单被退款时
   - 包含支付ID、退款金额等信息

#### 支付通知数据结构示例

```json
{
  "mch_id": "merchant123",
  "payment_id": "202405051234567890",
  "order_id": "order789",
  "user_id": "user456",
  "status": "PAID",
  "total_fee": "100.00",
  "order_type": "ONE_TIME",
  "event_type": "PAYMENT_SUCCESS",
  "paid_at": "2024-05-05T12:30:00Z",
  "tx_hash": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef"
}
```

### 订阅相关通知

订阅相关通知会在订阅状态发生变化或订阅账单处理时发送，包括以下事件类型：

1. **订阅成功通知** (event_type: SUBSCRIBE_SUCCESS)
   - 当用户成功授权订阅时
   - 包含订阅ID、授权地址等信息

2. **订阅失败通知** (event_type: SUBSCRIBE_FAILED)
   - 当订阅授权或初始化失败时
   - 包含订阅ID和失败原因

3. **订阅取消通知** (event_type: SUBSCRIBE_CANCELED)
   - 当用户取消订阅时
   - 包含订阅ID、已支付期数等信息

4. **订阅账单支付成功通知** (event_type: SUBSCRIBE_CHARGE_SUCCESS)
   - 当订阅周期性账单支付成功时
   - 包含订阅ID、账单期数、支付金额等信息

5. **订阅账单支付失败通知** (event_type: SUBSCRIBE_CHARGE_FAILED)
   - 当订阅周期性账单支付失败时
   - 包含订阅ID、账单期数、失败原因等信息

6. **订阅停止通知** (event_type: SUBSCRIBE_STOPPED)
   - 当订阅因余额不足或其他原因提前终止时
   - 包含订阅ID、停止原因等信息

#### 订阅通知数据结构示例

```json
{
  "mch_id": "merchant123",
  "subscribe_id": "202405051234567890",
  "order_id": "order789",
  "user_id": "user456",
  "status": "APPROVED",
  "event_type": "SUBSCRIBE_CHARGE_SUCCESS",
  "total_fee": "100.00",
  "subscriber_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
  "subscribe_paid_count": 3,
  "subscribe_total_count": 12,
  "subscribe_status": "ACTIVE",
  "tx_hash": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef"
}
```

## 认证与签名机制

### 统一认证机制
系统使用统一的中间件处理所有API请求的认证和签名验证。每个请求必须提供以下HTTP头：

- **X-MCH-ID**: 商户ID，用于识别请求来源
- **X-Timestamp**: 请求时间戳，格式为RFC3339（如：2023-01-01T12:00:00Z）
- **X-Nonce**: 随机字符串，用于防止重放攻击
- **X-Signature**: 请求签名，根据规范化方法计算的HMAC-SHA256哈希值

中间件会自动执行以下验证步骤：
1. 验证所有必需的头部信息是否存在
2. 验证时间戳是否在允许范围内（5分钟内）
3. 验证nonce未被使用过（防止重放攻击）
4. 根据请求参数计算规范化签名并验证是否匹配
5. 验证商户账户是否启用

### 规范化签名计算方法
所有API请求和通知的签名遵循以下统一的规范化规则：

#### API请求签名（使用GenerateCanonicalSignature）
规范化格式：`Method + Path + "?" + SortedCanonicalQueryString + "&body=" + CanonicalRequestBody + "&timestamp=" + timestamp + "&nonce=" + nonce + "&key=" + secretKey`

各部分说明：
- Method：HTTP方法（GET、POST等），大写
- Path：请求路径（如/api/v1/payments）
- SortedCanonicalQueryString：按键名字母排序并URL编码后的查询参数，格式为"key1=value1&key2=value2"
- CanonicalRequestBody：请求体的Base64编码，如果有请求体则前缀"&body="，如果没有请求体则省略整个body部分
- timestamp：时间戳（Unix秒数格式）
- nonce：随机字符串（16+字符，只包含十六进制字符）
- secretKey：商户密钥

#### 通知签名（使用GenerateCanonicalSignatureForNotification）
规范化格式：`"POST" + 通知URL + "&" + Base64(通知内容) + "&timestamp=" + timestamp + "&nonce=" + nonce + "&key=" + secretKey`

各部分说明：
- 通知URL：商户配置的通知接收URL
- 通知内容：完整的通知JSON的Base64编码
- timestamp：时间戳（Unix秒数格式）
- nonce：随机字符串
- secretKey：商户密钥

### 客户端集成指南

为简化API接入，系统提供了便捷的辅助方法，可帮助客户端应用轻松实现签名生成。

#### 方法一：使用SDK（推荐）

我们提供了多种语言的SDK，内置了规范化签名生成和验证功能。示例（Go语言）：

```go
// 创建API客户端
client := kiwi.NewClient("your-mch-id", "your-secret-key")

// 直接调用接口，SDK会自动生成和添加所有认证头
response, err := client.CreateOrder(CreateOrderRequest{
    OrderID:  "order123",
    UserID:   "user456",
    TotalFee: "100.00",
})
```

#### 方法二：使用辅助方法手动生成认证头

如果您需要自行实现API客户端，可以使用以下方法生成完整的认证头：

```go
// 创建请求体
requestBody, _ := json.Marshal(map[string]interface{}{
    "orderId":  "order123",
    "userId":   "user456",
    "totalFee": "100.00",
})

// 创建DigestService
digestService := NewDigestService()

// 生成所有认证头
headers := digestService.GenerateAuthHeaders(
    "POST",
    "/api/v1/payments",
    map[string]string{}, // 查询参数，如有需要可填充
    requestBody,
    "your-mch-id",
    "your-secret-key",
)

// 将headers添加到请求中
request, _ := http.NewRequest("POST", "https://api.example.com/api/v1/payments", bytes.NewBuffer(requestBody))
for key, value := range headers {
    request.Header.Set(key, value)
}
request.Header.Set("Content-Type", "application/json")

// 发送请求
client := &http.Client{}
response, err := client.Do(request)
```

#### 其他语言示例

**Python：**
```python
import requests
import json
import time
import base64
import hashlib
import hmac
import uuid
from datetime import datetime

def generate_canonical_signature(method, path, query_params, json_body, timestamp, nonce, secret_key):
    # 将查询参数按字母顺序排序
    sorted_query = "&".join(f"{k}={v}" for k, v in sorted(query_params.items())) if query_params else ""
    
    # 构建路径部分
    path_part = f"{path}?{sorted_query}" if sorted_query else path
    
    # 编码请求体
    encoded_body = ""
    if json_body:
        encoded_body = f"&body={base64.b64encode(json_body.encode()).decode()}"
    
    # 构建完整的规范化字符串
    canonical_string = f"{method}{path_part}{encoded_body}&timestamp={timestamp}&nonce={nonce}&key={secret_key}"
    
    # 计算HMAC-SHA256哈希
    signature = hmac.new(secret_key.encode(), canonical_string.encode(), hashlib.sha256).hexdigest()
    return signature

# 示例调用
mch_id = "your-mch-id"
secret_key = "your-secret-key"
method = "POST"
path = "/api/v1/payments"
query_params = {}  # 无查询参数
body = {
    "orderId": "order123",
    "userId": "user456",
    "totalFee": "100.00"
}
json_body = json.dumps(body)
timestamp = int(time.time())
nonce = str(uuid.uuid4())

# 生成签名
signature = generate_canonical_signature(
    method, path, query_params, json_body, timestamp, nonce, secret_key
)

# 准备请求头
headers = {
    "Content-Type": "application/json",
    "X-MCH-ID": mch_id,
    "X-Timestamp": datetime.utcfromtimestamp(timestamp).strftime('%Y-%m-%dT%H:%M:%SZ'),
    "X-Nonce": nonce,
    "X-Signature": signature
}

# 发送请求
response = requests.post(
    "https://api.example.com" + path,
    headers=headers,
    json=body
)
```

### API请求签名示例

HTTP请求：
```
POST /api/v1/payments?source=web HTTP/1.1
Host: api.example.com
X-MCH-ID: merchant123
X-Timestamp: 2023-01-01T12:00:00Z
X-Nonce: abc123
Content-Type: application/json

{
  "amount": "100.00",
  "userId": "user123"
}
```

1. 构造规范化签名原文：
   - Method: `POST`
   - Path: `/api/v1/payments`
   - SortedCanonicalQueryString: `source=web`
   - CanonicalRequestBody: `Base64({"amount":"100.00","userId":"user123"})`
     = `eyJhbW91bnQiOiIxMDAuMDAiLCJ1c2VySWQiOiJ1c2VyMTIzIn0=`
   - timestamp: `1672574400` (2023-01-01T12:00:00Z的Unix时间戳)
   - nonce: `abc123`
   - secretKey: `your-secret-key`

2. 完整的规范化字符串：
   `POST/api/v1/payments?source=web&body=eyJhbW91bnQiOiIxMDAuMDAiLCJ1c2VySWQiOiJ1c2VyMTIzIn0=&timestamp=1672574400&nonce=abc123&key=your-secret-key`

3. 计算HMAC-SHA256：
   签名 = HMAC-SHA256(规范化字符串, secretKey)

### 数据类型规范化规则

为确保一致的签名计算，不同数据类型的值将按以下规则进行规范化处理（仅用于传统的键值对签名方法）：

- 字符串：原样使用
- 整数：转换为十进制字符串，如 `42`
- 小数/浮点数：最多保留8位小数，移除尾部的0，如 `123.45`而非`123.45000`
- 布尔值：转换为字符串 `true` 或 `false`
- 时间：转换为RFC3339格式，如 `2023-01-01T12:00:00Z`
- 数组/对象：转换为JSON字符串
- 二进制数据：转换为Base64编码字符串
- null或undefined：转换为空字符串 `""`

**注意：** 在规范化签名方法中，整个请求体直接使用Base64编码，无需对单独的字段进行特殊处理。

### 重要注意事项

1. 签名计算中使用的是Unix秒级时间戳，而请求头中使用的是RFC3339格式时间戳
2. 对于请求体的签名，使用原始的JSON请求体进行Base64编码，不要对其进行任何修改或格式化
3. 所有API路径以相对路径形式提供（不包含域名或协议）
4. 查询参数必须按键名字母顺序排序
5. 每个请求的nonce必须唯一，建议使用UUID或其他随机值
6. 请根据文档提供的规范化方法实现签名生成和验证，不要使用已废弃的旧方法

### 请求体大小限制
所有API请求的请求体最大不应超过1MB。服务器会拒绝超出此限制的请求。

### 接口访问频率限制
为了保证服务稳定性，API接口有访问频率限制，基于请求来源IP地址针对特定路径前缀进行控制。超出限制的请求将收到 HTTP 429 (Too Many Requests) 错误。默认的限制如下：
- `/api/v1/payments` (支付相关接口): 每秒1个请求，允许短时突发60个请求。
- `/api/v1/subscribe` (订阅相关接口): 每秒1个请求，允许短时突发30个请求。
- `/pub/api/v1/` (公共API接口): 每秒20个请求，允许短时突发100个请求。

请合理控制请求频率，避免触发限制。

## 应用API后端接口

### 1. 根据商户订单号获取支付订单

接口: GET /api/v1/payments/order

描述: 根据商户订单号获取支付订单详情

参数:
- orderId: 商户订单号 (查询参数)

返回值:
- 成功: 支付订单详情，包括状态、金额、支付时间等
- 失败: 错误信息及原因，常见错误：
  - 支付订单不存在

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "202405051234567890",
    "mch_id": "merchant123",
    "user_id": "user456",
    "order_id": "order789",
    "total_fee": "100.00",
    "tax_fee": "10.00",
    "expire_at": "2024-05-05T13:00:00Z",
    "created_at": "2024-05-05T12:00:00Z",
    "status": "PENDING_PAY",
    "order_type": "ONE_TIME",
    "deposit_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
    "user_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F"
  },
  "systemTime": 1714899600000
}
```

### 2. 获取支付订单详情

接口: GET /api/v1/payments/get

描述: 根据支付ID获取支付订单详情

参数:
- id: 支付ID (查询参数)

返回值:
- 成功: 支付订单详情，包括状态、金额、支付时间等
- 失败: 错误信息及原因，常见错误：
  - 支付订单不存在

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "202405051234567890",
    "mch_id": "merchant123",
    "user_id": "user456",
    "order_id": "order789",
    "total_fee": "100.00",
    "tax_fee": "10.00",
    "expire_at": "2024-05-05T13:00:00Z",
    "created_at": "2024-05-05T12:00:00Z",
    "status": "PAID",
    "paid_at": "2024-05-05T12:30:00Z",
    "order_type": "ONE_TIME",
    "deposit_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
    "user_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F"
  },
  "systemTime": 1714899600000
}
```

### 3. 退款接口

接口: POST /api/v1/payments/refund

描述: 对已支付订单进行全额或部分退款

参数:
- id: 支付ID (查询参数)

请求体:
```json
{
  "refundAmount": "退款金额 - 必选，必须为正数"
}
```

验证要求:
- 必须提供有效的支付ID和退款金额
- 订单状态必须为已支付(PAID)
- 针对一次性订单(ONE_TIME)：退款金额不得超过订单总金额
- 针对订阅订单(SUBSCRIBE)：退款金额不得超过实际已支付总金额
- 一笔订单只能退款一次，退款后订单状态变为REFUNDED，不可再次退款

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "message": "Refund processed successfully",
    "amount": "50.00"
  },
  "systemTime": 1678234567890
}
```

错误返回示例:

```json
{
  "code": 0,
  "msg": "订单状态不是已支付状态",
  "data": null,
  "systemTime": 1678234567890
}
```

### 4. 创建订单

接口: POST /api/v1/payments

访问限制: 每分钟60次

请求体:

```json
{
  "orderId": "商户订单号 - 必选",
  "userId": "商户用户ID - 必选",
  "totalFee": "订单金额 - 必选",
  "taxFee": "税费金额 - 可选，默认为0，不得超过订单金额，已包含在totalFee中",
  "expireAt": "支付超时时间 - 可选，默认1小时",
  "memo": "订单备注 - 可选",
  "redirectURL": "支付成功后的页面跳转地址 - 可选",
  "logo": "订单图标地址 - 可选"
}
```

验证:
- 必须包含所有必选参数
- **注意**: 对于订阅类型(SUBSCRIBE)订单，请使用 `/api/v1/subscribe/create` 接口创建，本接口仅支持一次性(ONE_TIME)订单的创建。
- totalFee中已经包含taxFee，taxFee仅作为税费说明用途

返回值:
- 成功: 支付订单详情及钱包地址
- 失败: 错误信息及原因

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "202405051234567890",
    "mch_id": "merchant123",
    "user_id": "user456",
    "order_id": "order789",
    "total_fee": "100.00",
    "tax_fee": "10.00",
    "expire_at": "2024-05-05T13:00:00Z",
    "created_at": "2024-05-05T12:00:00Z",
    "status": "PENDING_PAY",
    "order_type": "ONE_TIME",
    "deposit_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
    "user_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F"
  },
  "systemTime": 1714899600000
}
```

# 订阅相关接口 - pull payment相关

**订阅功能不会自动扣款，只为应用系统提供扣款接口，具体的扣款周期和业务逻辑由应用自行实现**

1，创建订阅订单

- 引导用户跳到订阅页面，要求二选一
- 1、用户选择用账户余额支付 （从余额扣除，用户需要提前向地址充值）
- 2、用户选择链上支付（授权方式，只需要保证自动扣款前有足够余额）

2，扣款接口

- 应用要求系统进行扣款，如果是链上，金额会发送到支付系统预设地址中。（避免被黑，这个地址是在合约中的，需要有管理员私钥才能进行修改）
- 扣款成功/失败 回调应用系统，记录日志，成功才记录次数，失败不记录扣款次数

3，取消订阅

- 关闭这个订阅订单

4，修改订阅方式

- 任何时候引导用户跳转到1里面的二选一页面， 用户都可以重新选择支付方式,也可以取消订阅（页面会引导用户取消授权，同时通知应用服务取消订阅）

## 订阅相关API接口

### 1. 获取订阅详情

接口: GET /api/v1/subscribe/get

描述: 根据订阅ID获取订阅详情

参数:
- id: 订阅ID (查询参数)

验证要求:
- 必须提供有效的订阅ID
- 订阅必须属于当前商户

返回值:
- 成功: 订阅详情，包括费用、状态、支付期数等
- 失败: 错误信息及原因，常见错误：
  - 订阅不存在
  - 订阅不属于当前商户

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "202405051234567890",
    "mch_id": "merchant123",
    "user_id": "user456",
    "order_id": "order789",
    "fee": "100.00",
    "tax_fee": "10.00",
    "expire_at": "2024-05-05T13:00:00Z",
    "created_at": "2024-05-05T12:00:00Z",
    "status": "PENDING_SUBSCRIBE",
    "subscribe_total_count": 12,
    "subscribe_paid_count": 0,
    "subscribe_frequency": "Monthly",
    "deposit_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
    "user_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F"
  },
  "systemTime": 1714899600000
}
```

### 2. 创建订阅

接口: POST /api/v1/subscribe/create

描述: 创建新的订阅

请求体:
```json
{
  "orderId": "商户订单号 - 必选，最大64字符",
  "userId": "商户用户ID - 必选，最大64字符",
  "fee": "订阅费用 - 必选，必须为正数",
  "taxFee": "税费 - 可选，默认为0，不得超过费用，已包含在fee中",
  "expireAt": "过期时间 - 可选，默认为30天后",
  "memo": "备注 - 可选",
  "redirectURL": "跳转URL - 可选",
  "logo": "Logo URL - 可选",
  "subscribeTotalCount": "订阅总期数 - 必选，必须大于0",
  "subscribeFrequency": "订阅频率 - 可选，如'Monthly'、'Weekly'等"
}
```

验证要求:
- 必须包含所有必选参数
- 费用必须为正数
- 税费不得超过费用
- 订阅总期数必须大于0
- fee中已经包含taxFee，taxFee仅作为税费说明用途

返回值:
- 成功: 订阅详情及分配的钱包地址
- 失败: 错误信息及原因

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "202405051234567890",
    "mch_id": "merchant123",
    "user_id": "user456",
    "order_id": "order789",
    "fee": "100.00",
    "tax_fee": "10.00",
    "expire_at": "2024-06-04T12:00:00Z",
    "created_at": "2024-05-05T12:00:00Z",
    "status": "PENDING_SUBSCRIBE",
    "subscribe_total_count": 12,
    "subscribe_paid_count": 0,
    "subscribe_frequency": "Monthly",
    "deposit_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
    "user_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F"
  },
  "systemTime": 1714899600000
}
```

### 3. 创建订阅账单

接口: POST /api/v1/subscribe/bill/create

描述: 为指定的订阅创建一个收款账单，通常在每个收款周期由应用调用。系统会自动计算下一个账单期数，并自动取消任何之前未支付的账单。

请求体:
```json
{
  "subscribeId": "订阅ID - 必选",
  "billDate": "账单日期 - 可选，默认为当前时间",
  "dueDate": "账单到期日 - 可选，默认为当月最后一天",
  "fee": "账单金额 - 可选，默认使用订阅的费用，已包含税费"
}
```

验证要求:
- 必须提供有效的订阅ID
- 订阅必须属于当前商户
- 订阅的已付账单数量不得达到或超过订阅总期数
- 系统会自动计算下一个账单期数（所有现有账单中的最大期数 + 1）
- 若存在状态为"待支付"或"处理中"的账单，系统会自动将其状态设置为"已取消"
- 如果提供fee，则必须为正数，且已包含税费部分（税费比例与订阅一致）

返回值:
- 成功: 创建的订阅账单详情
- 失败: 错误信息及原因，常见错误：
  - 订阅不存在
  - 订阅不属于当前商户
  - 订阅已达到最大账单数量

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "202405051234567891",
    "subscribe_id": "202405051234567890",
    "mch_id": "merchant123",
    "user_id": "user456",
    "fee": "100.00",
    "status": "PENDING",
    "created_at": "2024-05-05T12:00:00Z",
    "bill_date": "2024-05-05T12:00:00Z",
    "due_date": "2024-05-31T23:59:59Z",
    "billing_period": 3
  },
  "systemTime": 1714899600000
}
```

### 4. 根据订阅ID获取订阅账单详情

接口: GET /api/v1/subscribe/bill/get

描述: 根据订阅账单ID获取账单详情

参数:
- id: 订阅账单ID (查询参数) - 必选

验证要求:
- 必须提供有效的订阅账单ID
- 订阅账单必须属于当前商户

返回值:
- 成功: 订阅账单详情，包括状态、费用、账单日期等
- 失败: 错误信息及原因，常见错误：
  - 账单不存在
  - 账单不属于当前商户

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "B202405051234567891",
    "subscribe_id": "S202405051234567890",
    "mch_id": "merchant123",
    "user_id": "user456",
    "fee": "100.00",
    "status": "PENDING",
    "created_at": "2024-05-05T12:00:00Z",
    "bill_date": "2024-05-05T12:00:00Z",
    "due_date": "2024-05-31T23:59:59Z",
    "billing_period": 3,
    "processed_at": null,
    "chain_id": null,
    "tx_hash": null
  },
  "systemTime": 1714899600000
}
```

### 10. 根据订阅ID和期数获取订阅账单详情

接口: GET /api/v1/subscribe/bill/getByOrderIDAndPeriod

描述: 根据商户订单ID和账单期数获取账单详情

参数:
- orderId: 商户订单ID (查询参数) - 必选
- period: 账单期数 (查询参数) - 必选

验证要求:
- 必须提供有效的订单ID和期数
- 订阅必须属于当前商户

返回值:
- 成功: 订阅账单详情
- 失败: 错误信息及原因，常见错误：
  - 订阅不存在
  - 指定期数的账单不存在
  - 订阅不属于当前商户

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "B202405051234567891",
    "subscribe_id": "S202405051234567890",
    "mch_id": "merchant123",
    "user_id": "user456",
    "fee": "100.00",
    "status": "PAID",
    "created_at": "2024-05-05T12:00:00Z",
    "bill_date": "2024-05-05T12:00:00Z",
    "due_date": "2024-05-31T23:59:59Z",
    "billing_period": 1,
    "processed_at": "2024-05-10T14:30:00Z",
    "chain_id": 1,
    "tx_hash": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef"
  },
  "systemTime": 1714899600000
}
```

### 11. 取消订阅

接口: POST /api/v1/subscribe/cancel

描述: 取消一个活跃的订阅

请求体:
```json
{
  "subscribeId": "订阅ID - 必选"
}
```

验证要求:
- 必须提供有效的订阅ID
- 订阅必须属于当前商户
- 订阅状态必须允许取消

返回值:
- 成功: 取消成功确认
- 失败: 错误信息及原因，常见错误：
  - 订阅不存在
  - 订阅状态不允许取消
  - 订阅不属于当前商户

返回示例:

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "message": "Subscription cancelled successfully",
    "subscribe_id": "S202405051234567890",
    "status": "CANCELED"
  },
  "systemTime": 1714899600000
}
```

