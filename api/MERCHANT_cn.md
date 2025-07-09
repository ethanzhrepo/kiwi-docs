# 商户接入指南 (MERCHANT_cn.md)

## 简介

本文档旨在帮助商户快速将其业务系统与Kiwi支付平台对接，实现一次性商品购买和周期性订阅服务收款。对接流程主要分为后端API调用和前端页面跳转两个部分。

## API认证

所有向Kiwi支付后端API发起的请求，都需要进行身份认证。详细的认证与签名机制，请参考 [应用API 接口概述 (REST_API_cn.md)](REST_API_cn.md#认证与签名机制)。您需要从Kiwi平台获取您的商户ID (`X-MCH-ID`) 和商户密钥 (`secretKey`) 用于API请求签名。

## 一次性支付接入流程

适用于用户单次购买商品或服务的场景。

### 1. 后端创建支付订单

您的后端服务首先需要调用Kiwi平台的"创建订单"接口，以生成一个支付订单。

**接口详情:**
- **Endpoint:** `POST /api/v1/payments`
- **文档参考:** [创建订单 (REST_API_cn.md)](REST_API_cn.md#4-创建订单)

**关键请求参数:**
- `orderId`: 您系统中的唯一订单号。
- `userId`: 您系统中的用户唯一标识。
- `totalFee`: 订单总金额（字符串，例如 "100.00"）。
- `taxFee` (可选): 税费金额（字符串，例如 "10.00"）。此金额已包含在 `totalFee` 中，仅作说明用途。默认为0。
- `expireAt` (可选): 支付订单的过期时间 (ISO 8601 格式，例如 "2023-12-31T23:59:59Z")。默认为1小时后。
- `memo` (可选): 订单备注信息（字符串）。
- `redirectURL` (可选): 用户支付完成后，Kiwi支付页面将会尝试跳转到此URL。具体用法见后文。
- `logo` (可选): 在Kiwi支付页面展示的品牌Logo URL（字符串）。

**示例请求体 (创建一次性支付订单):**
```json
{
  "orderId": "YOUR_UNIQUE_ORDER_ID_123",
  "userId": "YOUR_USER_ID_456",
  "totalFee": "99.99",
  "redirectURL": "https://yourdomain.com/payment/success?orderId=YOUR_UNIQUE_ORDER_ID_123"
}
```

**成功响应:**
接口会返回包含支付订单详情，其中最重要的字段是 `id` (Kiwi平台生成的支付订单ID，例如 "P2024072510300012345678") 和 `deposit_address` (收款地址)。您需要保存这些信息。

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "P2024072510300012345678", // Kiwi支付订单ID
    "mch_id": "your_mch_id",
    "user_id": "YOUR_USER_ID_456",
    "order_id": "YOUR_UNIQUE_ORDER_ID_123",
    "total_fee": "99.99",
    "tax_fee": "9.99", // 示例中传入的税费
    "created_at": "2024-07-25T10:30:00Z", // 订单创建时间
    "expire_at": "2024-07-25T11:30:00Z", // 订单过期时间
    "status": "PENDING_PAY",
    "order_type": "ONE_TIME", // 订单类型
    "deposit_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F", // 用户需要支付到的地址
    "memo": "这是一笔测试订单", // 备注信息
    "redirect_url": "https://yourdomain.com/payment/success?orderId=YOUR_UNIQUE_ORDER_ID_123", // 跳转URL
    "logo": "https://yourdomain.com/logo.png" // Logo URL
    // ... 其他可能的字段
  },
  "systemTime": 1721889000000
}
```

### 2. 前端页面集成：JWT生成与跳转

获取到Kiwi支付订单ID (`paymentId`)后，您需要在后端生成一个JWT，然后引导用户浏览器跳转到Kiwi支付的托管UI页面。

**UI支付页面路径:** `/payment/:paymentId`
例如: `https://<kiwi-ui-domain>/payment/P2024072510300012345678`

**JWT生成:**
JWT用于授权用户访问Kiwi支付UI页面并查看/操作其订单。
- **密钥:** 使用您的商户密钥 (`secretKey`) 进行签名。
- **算法:** HS256
- **Claims (声明):**
    - `iss` (Issuer): 您的商户ID (`mchId`)。
    - `aud` (Audience): 固定字符串 `"kiwi"`。
    - `userId`: 您系统中的用户ID (与创建订单时传入的 `userId` 一致)。
    - `exp` (Expiration Time): JWT的过期时间戳 (例如，当前时间 + 6小时)。
    - `paymentId` (自定义声明): Kiwi支付订单ID (从上一步获取的 `id`)。

详细JWT生成方法请参考后文 [JWT生成详解](#jwt生成详解)。

**跳转方式:**
将生成的JWT作为查询参数 `j` 附加到Kiwi支付UI页面URL上。

**示例跳转URL:**
`https://<kiwi-ui-domain>/payment/P2024072510300012345678?j=<GENERATED_JWT>`

Kiwi支付UI页面会解析此JWT，验证用户身份，并展示相应的支付信息。用户在UI页面完成支付操作。

## 订阅支付接入流程

适用于用户需要周期性支付的服务，例如会员订阅。

### 1. 后端创建订阅订单

您的后端服务调用Kiwi平台的"创建订阅"接口。

**接口详情:**
- **Endpoint:** `POST /api/v1/subscribe/create`
- **文档参考:** [创建订阅 (REST_API_cn.md)](REST_API_cn.md#2-创建订阅)

**关键请求参数:**
- `orderId`: 您系统中的唯一订单号。
- `userId`: 您系统中的用户唯一标识。
- `fee`: 每期订阅的费用（字符串，例如 "19.99"）。
- `taxFee` (可选): 每期税费金额（字符串，例如 "1.99"）。此金额已包含在 `fee` 中，仅作说明用途。默认为0。
- `subscribeTotalCount`: 订阅总期数 (整数, 必须大于0)。
- `subscribeFrequency` (可选): 订阅频率（字符串，例如 'Monthly', 'Weekly', 'Daily'）。
- `expireAt` (可选): 首次授权或支付的过期时间 (ISO 8601 格式)。默认为30天后。
- `memo` (可选): 订阅备注信息（字符串）。
- `redirectURL` (可选): 用户完成订阅授权/首期支付后，Kiwi订阅页面将会尝试跳转到此URL。
- `logo` (可选): 在Kiwi订阅页面展示的品牌Logo URL（字符串）。

**示例请求体:**
```json
{
  "orderId": "YOUR_SUBSCRIPTION_ORDER_ID_789",
  "userId": "YOUR_USER_ID_456",
  "fee": "19.99",
  "subscribeTotalCount": 12,
  "subscribeFrequency": "Monthly",
  "redirectURL": "https://yourdomain.com/subscription/activated?subId=YOUR_SUBSCRIPTION_ORDER_ID_789"
}
```

**成功响应:**
接口会返回包含订阅详情，其中最重要的字段是 `id` (Kiwi平台生成的订阅ID，例如 "S2024072511000098765432") 和 `deposit_address` (首期支付/后续如需手动支付的收款地址)。您需要保存这些信息。

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "S2024072511000098765432", // Kiwi订阅ID
    "mch_id": "your_mch_id",
    "user_id": "YOUR_USER_ID_456",
    "order_id": "YOUR_SUBSCRIPTION_ORDER_ID_789",
    "fee": "19.99",
    "tax_fee": "1.99", // 示例中传入的税费
    "subscribe_total_count": 12,
    "subscribe_paid_count": 0, // 初始已支付期数为0
    "subscribe_frequency": "Monthly", // 订阅频率
    "created_at": "2024-07-25T11:00:00Z", // 创建时间
    "expire_at": "2024-08-24T11:00:00Z", // 授权/首期支付过期时间
    "status": "PENDING_SUBSCRIBE", // 初始状态
    "deposit_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F", // 收款地址
    "memo": "月度会员订阅", // 备注
    "redirect_url": "https://yourdomain.com/subscription/activated?subId=YOUR_SUBSCRIPTION_ORDER_ID_789", // 跳转URL
    "logo": "https://yourdomain.com/logo.png" // Logo URL
    // ... 其他可能的字段
  },
  "systemTime": 1721890800000
}
```

### 2. 前端页面集成：JWT生成与跳转

获取到Kiwi订阅ID (`subscribeId`)后，流程与一次性支付类似：后端生成JWT，然后引导用户跳转。

**UI订阅页面路径:** `/subscribe/:subscribeId`
例如: `https://<kiwi-ui-domain>/subscribe/S2024072511000098765432`

**JWT生成:**
- **密钥:** 使用您的商户密钥 (`secretKey`)。
- **算法:** HS256
- **Claims (声明):**
    - `iss` (Issuer): 您的商户ID (`mchId`)。
    - `aud` (Audience): 固定字符串 `"kiwi"`。
    - `userId`: 您系统中的用户ID (与创建订阅时传入的 `userId` 一致)。
    - `exp` (Expiration Time): JWT的过期时间戳。
    - `subscribeId` (自定义声明): Kiwi订阅ID (从上一步获取的 `id`)。

**跳转方式:**
将生成的JWT作为查询参数 `j` 附加到Kiwi订阅UI页面URL上。

**示例跳转URL:**
`https://<kiwi-ui-domain>/subscribe/S2024072511000098765432?j=<GENERATED_JWT>`

用户将在Kiwi订阅UI页面完成授权或首期支付操作。

## JWT生成详解

在将用户重定向到Kiwi支付/订阅UI页面前，您的后端必须生成一个JWT（JSON Web Token）。这个JWT用于向Kiwi前端UI证明用户请求的合法性。

**JWT结构:**
JWT由三部分组成：Header, Payload (Claims), Signature。

1.  **Header:**
    ```json
    {
      "alg": "HS256",
      "typ": "JWT"
    }
    ```
2.  **Payload (Claims):**
    - `iss` (Issuer): string - 您的商户ID (`mchId`)。
    - `aud` (Audience): string - 固定为 `"kiwi"`。
    - `userId`: string - 您系统中的用户唯一标识。**此ID必须与创建订单/订阅时传入的`userId`完全匹配。**
    - `exp` (Expiration Time): numeric - Unix时间戳，表示JWT的过期时间。建议设置为一个合理的值，例如当前时间后6小时。
    - `paymentId` (仅一次性支付需要): string - Kiwi支付订单ID。
    - `subscribeId` (仅订阅支付需要): string - Kiwi订阅ID。

    **重要:**
    - 对于一次性支付流程，请包含 `paymentId`。
    - 对于订阅支付流程，请包含 `subscribeId`。不要同时包含两者。

3.  **Signature:**
    使用HS256算法，将编码后的Header和Payload，通过您的商户密钥 (`secretKey`)进行签名。

**示例 (伪代码 - Python 使用 PyJWT库):**
```python
import jwt
import time

# 您的商户信息
mch_id = "YOUR_MCH_ID"
secret_key = "YOUR_SECRET_KEY" # 这是您的商户密钥
user_id_from_your_system = "USER_XYZ789"

# 从Kiwi API获取的ID
kiwi_payment_id = "P2024072510300012345678" # 或 kiwi_subscribe_id

# 设置JWT过期时间 (例如：6小时后)
expiration_time = int(time.time()) + (6 * 60 * 60)

# 构建Payload
payload = {
    "iss": mch_id,
    "aud": "kiwi",
    "userId": user_id_from_your_system,
    "exp": expiration_time
}

# 根据流程类型添加特定ID
is_one_time_payment = True # 或 False for subscription
if is_one_time_payment:
    payload["paymentId"] = kiwi_payment_id
else:
    payload["subscribeId"] = kiwi_subscribe_id # 使用对应的订阅ID

# 生成JWT
encoded_jwt = jwt.encode(payload, secret_key, algorithm="HS256")

# encoded_jwt 即为需要传递给前端的token
# print(encoded_jwt)
```

**注意:**
- 请务必使用从Kiwi平台获取的官方商户密钥。
- 确保`userId`的准确性，这是授权用户访问其订单/订阅的关键。
- 许多编程语言都有成熟的JWT库可以简化生成过程（如Java的 `jjwt`，Node.js的 `jsonwebtoken` 等）。

## `redirectURL` 使用说明

在调用"创建订单"或"创建订阅"接口时，您可以传入 `redirectURL` 参数。Kiwi支付/订阅UI在用户完成相应操作（如支付成功、授权成功）后，会尝试将用户浏览器重定向到此 `redirectURL`。

**两种主要场景:**

1.  **跳转回商户指定页面:**
    如果您希望用户在Kiwi平台完成操作后，返回到您应用的特定页面（例如订单成功页、会员中心），您可以将该页面的完整URL作为 `redirectURL`。
    ```
    "redirectURL": "https://yourdomain.com/payment/thankyou?orderId=YOUR_ORDER_ID"
    ```
    Kiwi平台会将用户浏览器重定向到 `https://yourdomain.com/payment/thankyou?orderId=YOUR_ORDER_ID`。
    您的页面可以解析URL中的参数（如`orderId`）来展示相应的信息。
    建议在此 `redirectURL` 对应的页面中，再次调用您的后端接口查询订单/订阅的最新状态，以确保信息准确性，而不是仅依赖于前端跳转。

2.  **用户关闭支付/订阅窗口:**
    如果您希望用户完成操作后直接关闭Kiwi支付/订阅的窗口或标签页（常见于弹窗或新标签页打开支付的场景），您可以将 `redirectURL` 设置为一个特殊值：`kiwi://close`。
    ```
    "redirectURL": "kiwi://close"
    ```
    当Kiwi前端UI检测到此 `redirectURL` 时，它会尝试执行 `window.close()` 操作。
    **请注意:** 浏览器对于 `window.close()` 的行为有安全限制。通常，只有由脚本打开的窗口才能被脚本关闭。如果Kiwi支付页面不是由您的脚本（例如 `window.open()`）打开的，`kiwi://close` 可能不会按预期关闭窗口，用户可能需要手动关闭。

**不提供 `redirectURL`:**
如果您不提供 `redirectURL`，用户在完成操作后，将停留在Kiwi平台的默认结果页面。

## 高级集成：直接展示收款地址 (不跳转Kiwi UI)

对于希望更深度定制用户体验，或不希望/不需要用户跳转到Kiwi支付页面的商户，可以选择直接向用户展示收款地址的集成方式。此方式尤其适用于API驱动的流程或特定支付场景。

**流程概述:**

1.  **后端创建订单/订阅:**
    您的后端服务正常调用Kiwi平台的"创建订单" (`POST /api/v1/payments`) 或"创建订阅" (`POST /api/v1/subscribe/create`) 接口。
    详细参数请参考前文对应章节。

2.  **获取收款地址:**
    从API的成功响应中，提取 `data.deposit_address` 字段。这是用户需要实际支付到的加密货币地址。
    对于一次性支付，这是唯一的支付地址。
    对于订阅，这通常是首期支付或后续手动支付的地址（如果订阅不是自动链上扣款类型或用户选择余额支付）。

3.  **商户前端展示地址:**
    您的应用系统（网站、App等）直接向用户展示此 `deposit_address` 以及期望支付的金额和币种（通常是USDT等稳定币，具体取决于您的配置和Kiwi平台支持）。
    您可以同时展示一个二维码方便用户扫码支付。

    **重要提示:**
    - **明确告知用户支付金额和币种。**
    - **提示用户支付网络/链** (例如 Ethereum (ERC20), TRON (TRC20)等)，确保用户不会转错链导致资金损失。Kiwi平台通常会为特定币种在特定链上分配地址。
    - **设置合理的支付时效提醒**，因为订单本身可能有过期时间 (`expireAt`)。

4.  **用户自行完成支付:**
    用户通过自己的钱包向展示的 `deposit_address` 转入相应金额的加密货币。

5.  **依赖后端通知确认支付:**
    在此集成模式下，**您系统的订单状态更新完全依赖于Kiwi平台发送的后端异步通知** (Webhook)。当Kiwi平台监测到对应地址收到款项并确认后，会向您配置的通知URL发送支付成功或订阅状态更新的通知。
    - 关于通知的接收和处理，请严格遵循 [应用API 接口概述 (REST_API_cn.md)](REST_API_cn.md#通知机制) 中的说明，特别是签名验证和幂等性处理。

**与跳转Kiwi UI方式的对比:**

-   **用户体验:** 商户完全控制用户界面，用户无需离开商户平台。
-   **前端集成:** 无需处理JWT生成和前端跳转逻辑，也无需关心 `token` 查询参数。
-   **`redirectURL`:** 在此模式下，后端创建订单时传递的 `redirectURL` 将不会被用于用户界面的跳转，因为用户根本不访问Kiwi的UI。
-   **支付状态更新:** 严格依赖后端Webhook通知。商户需要自行实现用户侧的支付状态轮询或等待提示。
-   **适用场景:** 对UI有高度定制化需求、API驱动的自动化流程、不希望用户跳转的内嵌式支付等。

**注意事项:**
- 由于用户直接向地址付款，商户需要有良好的机制提示用户支付的准确金额、币种和网络，避免用户操作失误。
- 强烈建议在您的用户界面清晰展示订单的支付状态，并在收到Kiwi后端通知后及时更新。

## 支付通知

无论用户是否通过 `redirectURL` 返回您的网站，Kiwi平台都会通过后端异步通知机制，将支付或订阅状态的变更通知到您在Kiwi商户后台配置的"通知URL"。

请务必依赖此异步通知来更新您系统中的订单状态，因为它比前端跳转更为可靠。

### 通知头部信息

Kiwi平台发送的通知请求会包含以下HTTP头：

-   **`Content-Type`**: `application/json;charset=utf-8`
-   **`X-KIWI-SIGN`**: 通知内容的签名，用于验证通知的真实性。
-   **`X-KIWI-TIMESTAMP`**: 请求时间戳 (Unix秒数格式，例如 `1678234567`)。
-   **`X-KIWI-NONCE`**: 随机字符串，用于防止重放攻击。

### 通知签名验证

商户应通过以下步骤验证通知的真实性，以确保通知来自Kiwi平台且内容未被篡改：

1.  **准备参与签名的参数：**
    *   `notifyURL`: 您在Kiwi商户后台配置的接收通知的完整URL。
    *   `requestBody`: 从HTTP POST请求中获取的原始JSON通知内容 (字节流)。
    *   `timestamp`: 从 `X-KIWI-TIMESTAMP` 请求头中获取的时间戳字符串。
    *   `nonce`: 从 `X-KIWI-NONCE` 请求头中获取的随机字符串。
    *   `secretKey`: 您在Kiwi平台获取的商户密钥。

2.  **构建规范化字符串 (Canonical String):**
    将上述参数按照以下格式拼接：
    `"POST" + notifyURL + "&" + Base64Encode(requestBody) + "&timestamp=" + timestamp + "&nonce=" + nonce + "&key=" + secretKey`

    *   `POST`: HTTP方法，固定为大写POST。
    *   `Base64Encode(requestBody)`: 对原始JSON通知内容 (字节流) 进行Base64编码。

3.  **计算签名:**
    对构建的规范化字符串使用商户密钥计算HMAC-SHA256哈希值，然后将哈希结果转换为十六进制字符串。

4.  **比较签名:**
    将计算得到的签名与 `X-KIWI-SIGN` 请求头中的值进行比较。如果两者一致，则签名验证通过。

**示例代码 (Go):**
```go
import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/base64"
    "encoding/hex"
    "fmt"
    "io/ioutil"
    "net/http"
    "sort"
    "strings"
    "time"
)

// verifyKiwiNotificationSignature 验证Kiwi通知签名
func verifyKiwiNotificationSignature(r *http.Request, secretKey string) (bool, error) {
    // 获取必要的头部信息
    kiwiSign := r.Header.Get("X-KIWI-SIGN")
    kiwiTimestamp := r.Header.Get("X-KIWI-TIMESTAMP")
    kiwiNonce := r.Header.Get("X-KIWI-NONCE")

    if kiwiSign == "" || kiwiTimestamp == "" || kiwiNonce == "" {
        return false, fmt.Errorf("missing required Kiwi notification headers")
    }

    // 校验时间戳 (例如，5分钟内有效)
    timestampInt, err := strconv.ParseInt(kiwiTimestamp, 10, 64)
    if err != nil {
        return false, fmt.Errorf("invalid X-KIWI-TIMESTAMP format")
    }
    if time.Now().Unix()-timestampInt > 300 { // 5 minutes tolerance
        // return false, fmt.Errorf("X-KIWI-TIMESTAMP expired")
        // 注意：对于通知，时间戳过期校验可以更宽松或根据业务调整，因为重试可能导致延迟
    }

    // 读取请求体
    requestBody, err := ioutil.ReadAll(r.Body)
    if err != nil {
        return false, fmt.Errorf("failed to read request body: %w", err)
    }
    // 重新填充Body，以便后续处理（如果需要）
    r.Body = ioutil.NopCloser(bytes.NewBuffer(requestBody))

    // 构建规范化字符串
    // notifyURL 是您配置的接收通知的URL
    // 在实际应用中，您需要知道这个URL
    notifyURL := r.URL.String() // 这通常是请求的路径，您可能需要拼接上域名等得到完整URL
                                // 或者直接使用您在Kiwi后台配置的那个URL字符串
    
    encodedBody := base64.StdEncoding.EncodeToString(requestBody)
    
    var parts []string
    parts = append(parts, "POST"+notifyURL)
    parts = append(parts, encodedBody) // body是base64编码后的
    parts = append(parts, "timestamp="+kiwiTimestamp)
    parts = append(parts, "nonce="+kiwiNonce)
    parts = append(parts, "key="+secretKey)

    // 使用 & 符号连接各部分
    canonicalString := strings.Join(parts, "&")
    
    // 计算HMAC-SHA256哈希
    mac := hmac.New(sha256.New, []byte(secretKey))
    mac.Write([]byte(canonicalString))
    calculatedSignature := hex.EncodeToString(mac.Sum(nil))
    
    // 比较签名
    return calculatedSignature == kiwiSign, nil
}
```
**注意:** 上述Go示例中的 `notifyURL` 部分需要您根据实际情况替换为您在Kiwi后台配置的接收通知的URL。

### 通知处理要求

1.  **幂等性处理:** 由于网络原因或商户服务器未能及时响应，Kiwi平台可能会重复发送同一条通知。商户系统必须能够正确处理重复通知，确保业务逻辑只执行一次。建议通过检查通知中的唯一标识（如 `payment_or_subscribe_id` 和 `event_type` 的组合）来判断通知是否已处理。
2.  **响应确认:** 接收到通知后，商户服务器必须返回HTTP状态码 `200` (OK) 以确认成功接收。如果返回其他任何状态码（包括网络超时），Kiwi平台会认为通知失败，并会按照预设策略进行重试。
3.  **重试策略:** Kiwi平台对发送失败的通知会进行多次重试。重试间隔大致如下：
    *   首次失败后：1分钟
    *   之后：5分钟、10分钟、30分钟
    *   然后：每1小时重试一次
    *   最长重试时间：通常为24小时内。
    具体重试次数和间隔可能会根据系统配置调整。
4.  **及时处理:** 商户应尽快处理通知并返回响应，避免因处理超时导致Kiwi平台误判为失败并发起重试。建议将耗时操作异步处理。

### 通知类型 (Event Types)

Kiwi平台会针对不同的业务事件发送通知。主要的事件类型 (`event_type`) 如下，通知内容会包含与该事件相关的核心信息：

**重要通知内容说明:**
当前版本的通知服务发送的JSON负载主要包含以下核心身份识别字段。商户在收到通知后，应使用这些字段（特别是 `payment_or_subscribe_id` 和 `event_type`）通过API查询详细的订单/订阅/账单状态和信息，而不是期望通知本身包含所有细节数据。

**通知包含的核心字段:**
- `mch_id`: 商户ID。
- `user_id`: 用户ID。
- `order_id`: 商户原始订单号。
- `payment_or_subscribe_id`: Kiwi平台的支付ID、订阅ID或账单ID，具体取决于事件上下文。
- `type`: 通知类型，`ONE-TIME` 或 `SUBSCRIBE`。
- `event_type`: 当前的事件类型。

**1. 一次性支付相关通知 (`type: "ONE-TIME"`)**

-   **`PAYMENT_SUCCESS`**: 支付成功。
-   **`PAYMENT_FAILED`**: 支付失败。
-   **`PAYMENT_CLOSED`**: 订单被关闭 (例如手动关闭)。
-   **`PAYMENT_REFUNDED`**: 订单已退款。

**2. 订阅相关通知 (`type: "SUBSCRIBE"`)**

-   **`SUBSCRIBE_SUCCESS`**: 订阅创建/授权成功。
    *   触发场景: 用户通过UI确认待处理的订阅 (调用 `/pub/api/v1/user/subscribe/confirm/:id`) 或 用户对已是 `SUBSCRIBED` 状态的订阅完成链上授权 (调用 `/pub/api/v1/user/personalSign/:id`)。
    *   `payment_or_subscribe_id` 字段为此订阅的 `subscribe.ID`。
-   **`SUBSCRIBE_CANCELED`**: 订阅已被取消。
    *   触发场景: 商户通过API (`/api/v1/subscribe/cancel`) 或用户通过UI (`/pub/api/v1/user/subscribe/cancel/:id`) 主动取消订阅。
    *   `payment_or_subscribe_id` 字段为此订阅的 `subscribe.ID`。
-   **`SUBSCRIBE_CHARGE_SUCCESS`**: 订阅周期扣款成功。
    *   触发场景: 系统通过余额成功处理周期性账单 (由 `/api/v1/subscribe/bill/create` 后，如用户余额充足则自动触发，或后续由 `ChargeBill` 服务调用) 或 链上支付的订阅账单被确认成功 (由 `SubscribeCheckJob` 处理)。
    *   `payment_or_subscribe_id` 字段为此成功的 `bill.ID`。
-   **`SUBSCRIBE_CHARGE_FAILED`**: 订阅周期扣款失败。
    *   触发场景: 链上支付的订阅账单被确认失败或长时间未确认 (由 `SubscribeCheckJob` 处理)。
    *   注意: 当前并非所有扣款失败场景都会触发此通知 (例如，通过余额扣款时若余额不足且未配置链上支付，或 `ChargeBill` 服务在尝试链上支付前检测到配置问题，可能不会发送此通知)。
    *   `payment_or_subscribe_id` 字段为此失败的 `bill.ID`。
-   **`SUBSCRIBE_STOPPED`**: 订阅因异常情况停止。
    *   触发场景: 商户通过API (`/api/v1/subscribe/stop`) 停止订阅 (订阅状态会变为 `CANCELED`)。
    *   `payment_or_subscribe_id` 字段为此订阅的 `subscribe.ID`。

### 通知数据结构示例

以下为一些常见通知类型的JSON数据结构示例。请再次注意，实际通知内容仅包含核心字段，商户应使用这些ID通过API获取详细信息。

**支付成功 (PAYMENT_SUCCESS) 示例:**
```json
{
  "mch_id": "merchant123",
  "user_id": "user456",
  "order_id": "order_abc_1001",
  "payment_or_subscribe_id": "P2024072610300012345678",
  "type": "ONE-TIME",
  "event_type": "PAYMENT_SUCCESS",
  "total_fee": "99.99",
  "paid_at": "2024-07-26T10:35:00Z",
  "tx_hash": "0xabcdef1234567890..." // 如果是链上支付
}
```

**订阅扣款成功 (SUBSCRIBE_CHARGE_SUCCESS) 示例:**
```json
{
  "mch_id": "merchant123",
  "user_id": "user789",
  "order_id": "sub_order_xyz_2002", 
  "payment_or_subscribe_id": "B2024082611050055544332", // 这是 bill.ID
  "type": "SUBSCRIBE",
  "event_type": "SUBSCRIBE_CHARGE_SUCCESS"
}
```

**订阅扣款失败 (SUBSCRIBE_CHARGE_FAILED) 示例:**
```json
{
  "mch_id": "merchant123",
  "user_id": "user789",
  "order_id": "sub_order_xyz_2002", 
  "payment_or_subscribe_id": "B2024082611050055544333", // 这是 bill.ID
  "type": "SUBSCRIBE",
  "event_type": "SUBSCRIBE_CHARGE_FAILED"
}
```

**订阅停止 (SUBSCRIBE_STOPPED) 示例:**
```json
{
  "mch_id": "merchant123",
  "user_id": "user789",
  "order_id": "sub_order_xyz_2002", 
  "payment_or_subscribe_id": "S2024072611000098765432", // 这是 subscribe.ID
  "type": "SUBSCRIBE",
  "event_type": "SUBSCRIBE_STOPPED"
}
```

---
如有疑问，请联系Kiwi支付技术支持。 

### 3. 创建订阅账单

接口: POST /api/v1/subscribe/bill/create

描述: 为指定的订阅创建一个收款账单，通常在每个收款周期由应用调用。系统会自动计算下一个账单期数，并自动取消任何之前未支付的账单。

请求体:
```json
{
  "subscribeId": "订阅ID - 必选",
  "billDate": "账单日期 - 可选，默认为当前时间",
  "dueDate": "账单到期日 - 可选，默认为当月最后一天", 
  "fee": "账单金额 - 可选，默认使用订阅的费用"
}
```

**重要说明**:
- 系统会自动计算下一个账单期数（所有现有账单中的最大期数 + 1）
- 如果存在状态为"待支付"或"处理中"的账单，系统会自动将其状态设置为"已取消"，以确保每个订阅在同一时间只有一个活跃账单
- 此设计允许商户在任何需要收款的时间点创建订阅账单，而不必担心期数管理 