# Merchant Integration Guide (MERCHANT_en.md)

## Introduction

This document helps merchants quickly integrate their business systems with the Kiwi payment platform, enabling one-time product purchases and periodic subscription service payments. The integration process is mainly divided into two parts: backend API calls and frontend page redirections.

## API Authentication

All requests to the Kiwi payment backend API require authentication. For detailed authentication and signature mechanisms, please refer to [Application API Overview (REST_API_en.md)](REST_API_en.md#authentication-and-signature-mechanism). You'll need to obtain your merchant ID (`X-MCH-ID`) and merchant key (`secretKey`) from the Kiwi platform for API request signing.

## One-Time Payment Integration Process

Applicable for scenarios where users purchase products or services once.

### 1. Backend Payment Order Creation

Your backend service first needs to call the Kiwi platform's "Create Order" interface to generate a payment order.

**Interface Details:**
- **Endpoint:** `POST /api/v1/payments`
- **Documentation Reference:** [Create Order (REST_API_en.md)](REST_API_en.md#4-create-order)

**Key Request Parameters:**
- `orderId`: The unique order number in your system.
- `userId`: The unique user identifier in your system.
- `totalFee`: Total order amount (string, e.g., "100.00").
- `taxFee` (optional): Tax amount (string, e.g., "10.00"). This amount is already included in `totalFee` and is provided for informational purposes only. Default is 0.
- `expireAt` (optional): Payment order expiration time (ISO 8601 format, e.g., "2023-12-31T23:59:59Z"). Default is 1 hour later.
- `memo` (optional): Order note information (string).
- `redirectURL` (optional): After the user completes payment, the Kiwi payment page will attempt to redirect to this URL. See details below.
- `logo` (optional): Brand Logo URL displayed on the Kiwi payment page (string).

**Example Request Body (Creating a One-Time Payment Order):**
```json
{
  "orderId": "YOUR_UNIQUE_ORDER_ID_123",
  "userId": "YOUR_USER_ID_456",
  "totalFee": "99.99",
  "redirectURL": "https://yourdomain.com/payment/success?orderId=YOUR_UNIQUE_ORDER_ID_123"
}
```

**Successful Response:**
The interface will return payment order details, the most important fields being `id` (Kiwi platform-generated payment order ID, e.g., "P2024072510300012345678") and `deposit_address` (receiving address). You need to save this information.

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "P2024072510300012345678", // Kiwi payment order ID
    "mch_id": "your_mch_id",
    "user_id": "YOUR_USER_ID_456",
    "order_id": "YOUR_UNIQUE_ORDER_ID_123",
    "total_fee": "99.99",
    "tax_fee": "9.99", // Example tax fee provided
    "created_at": "2024-07-25T10:30:00Z", // Order creation time
    "expire_at": "2024-07-25T11:30:00Z", // Order expiration time
    "status": "PENDING_PAY",
    "order_type": "ONE_TIME", // Order type
    "deposit_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F", // Address where user needs to pay
    "memo": "This is a test order", // Note information
    "redirect_url": "https://yourdomain.com/payment/success?orderId=YOUR_UNIQUE_ORDER_ID_123", // Redirect URL
    "logo": "https://yourdomain.com/logo.png" // Logo URL
    // ... other possible fields
  },
  "systemTime": 1721889000000
}
```

### 2. Frontend Integration: JWT Generation and Redirection

After obtaining the Kiwi payment order ID (`paymentId`), you need to generate a JWT in your backend, then guide the user's browser to redirect to Kiwi payment's hosted UI page.

**UI Payment Page Path:** `/payment/:paymentId`
Example: `https://<kiwi-ui-domain>/payment/P2024072510300012345678`

**JWT Generation:**
JWT is used to authorize user access to the Kiwi payment UI page and view/operate their orders.
- **Key:** Use your merchant key (`secretKey`) for signing.
- **Algorithm:** HS256
- **Claims:**
    - `iss` (Issuer): Your merchant ID (`mchId`).
    - `aud` (Audience): Fixed string `"kiwi"`.
    - `userId`: Your system's user ID (consistent with the `userId` passed when creating the order).
    - `exp` (Expiration Time): JWT's expiration timestamp (e.g., current time + 6 hours).
    - `paymentId` (custom claim): Kiwi payment order ID (the `id` obtained in the previous step).

For detailed JWT generation methods, please refer to [JWT Generation Details](#jwt-generation-details) below.

**Redirection Method:**
Append the generated JWT as a query parameter `j` to the Kiwi payment UI page URL.

**Example Redirection URL:**
`https://<kiwi-ui-domain>/payment/P2024072510300012345678?j=<GENERATED_JWT>`

The Kiwi payment UI page will parse this JWT, verify the user's identity, and display the corresponding payment information. The user completes the payment operation on the UI page.

## Subscription Payment Integration Process

Applicable for services requiring periodic payments, such as membership subscriptions.

### 1. Backend Subscription Order Creation

Your backend service calls the Kiwi platform's "Create Subscription" interface.

**Interface Details:**
- **Endpoint:** `POST /api/v1/subscribe/create`
- **Documentation Reference:** [Create Subscription (REST_API_en.md)](REST_API_en.md#2-create-subscription)

**Key Request Parameters:**
- `orderId`: The unique order number in your system.
- `userId`: The unique user identifier in your system.
- `fee`: Cost per subscription period (string, e.g., "19.99").
- `taxFee` (optional): Tax amount per period (string, e.g., "1.99"). This amount is already included in `fee` and is provided for informational purposes only. Default is 0.
- `subscribeTotalCount`: Total subscription periods (integer, must be greater than 0).
- `subscribeFrequency` (optional): Subscription frequency (string, e.g., 'Monthly', 'Weekly', 'Daily').
- `expireAt` (optional): First authorization or payment expiration time (ISO 8601 format). Default is 30 days later.
- `memo` (optional): Subscription note information (string).
- `redirectURL` (optional): After the user completes subscription authorization/initial payment, the Kiwi subscription page will attempt to redirect to this URL.
- `logo` (optional): Brand Logo URL displayed on the Kiwi subscription page (string).

**Example Request Body:**
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

**Successful Response:**
The interface will return subscription details, the most important fields being `id` (Kiwi platform-generated subscription ID, e.g., "S2024072511000098765432") and `deposit_address` (initial payment/subsequent manual payment receiving address if needed). You need to save this information.

```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "S2024072511000098765432", // Kiwi subscription ID
    "mch_id": "your_mch_id",
    "user_id": "YOUR_USER_ID_456",
    "order_id": "YOUR_SUBSCRIPTION_ORDER_ID_789",
    "fee": "19.99",
    "tax_fee": "1.99", // Example tax fee provided
    "subscribe_total_count": 12,
    "subscribe_paid_count": 0, // Initial paid periods is 0
    "subscribe_frequency": "Monthly", // Subscription frequency
    "created_at": "2024-07-25T11:00:00Z", // Creation time
    "expire_at": "2024-08-24T11:00:00Z", // Authorization/initial payment expiration time
    "status": "PENDING_SUBSCRIBE", // Initial status
    "deposit_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F", // Receiving address
    "memo": "Monthly membership subscription", // Note
    "redirect_url": "https://yourdomain.com/subscription/activated?subId=YOUR_SUBSCRIPTION_ORDER_ID_789", // Redirect URL
    "logo": "https://yourdomain.com/logo.png" // Logo URL
    // ... other possible fields
  },
  "systemTime": 1721890800000
}
```

### 2. Frontend Integration: JWT Generation and Redirection

After obtaining the Kiwi subscription ID (`subscribeId`), the process is similar to one-time payment: backend generates JWT, then guides the user to redirect.

**UI Subscription Page Path:** `/subscribe/:subscribeId`
Example: `https://<kiwi-ui-domain>/subscribe/S2024072511000098765432`

**JWT Generation:**
- **Key:** Use your merchant key (`secretKey`).
- **Algorithm:** HS256
- **Claims:**
    - `iss` (Issuer): Your merchant ID (`mchId`).
    - `aud` (Audience): Fixed string `"kiwi"`.
    - `userId`: Your system's user ID (consistent with the `userId` passed when creating the subscription).
    - `exp` (Expiration Time): JWT's expiration timestamp.
    - `subscribeId` (custom claim): Kiwi subscription ID (the `id` obtained in the previous step).

**Redirection Method:**
Append the generated JWT as a query parameter `j` to the Kiwi subscription UI page URL.

**Example Redirection URL:**
`https://<kiwi-ui-domain>/subscribe/S2024072511000098765432?j=<GENERATED_JWT>`

The user will complete authorization or initial payment on the Kiwi subscription UI page.

## JWT Generation Details

Before redirecting users to the Kiwi payment/subscription UI page, your backend must generate a JWT (JSON Web Token). This JWT is used to prove the legitimacy of the user's request to the Kiwi frontend UI.

**JWT Structure:**
JWT consists of three parts: Header, Payload (Claims), Signature.

1.  **Header:**
    ```json
    {
      "alg": "HS256",
      "typ": "JWT"
    }
    ```
2.  **Payload (Claims):**
    - `iss` (Issuer): string - Your merchant ID (`mchId`).
    - `aud` (Audience): string - Fixed as `"kiwi"`.
    - `userId`: string - The unique user identifier in your system. **This ID must exactly match the `userId` passed when creating the order/subscription.**
    - `exp` (Expiration Time): numeric - Unix timestamp, indicating JWT's expiration time. Recommended to set a reasonable value, such as 6 hours after the current time.
    - `paymentId` (only required for one-time payment): string - Kiwi payment order ID.
    - `subscribeId` (only required for subscription payment): string - Kiwi subscription ID.

    **Important:**
    - For one-time payment flow, include `paymentId`.
    - For subscription payment flow, include `subscribeId`. Do not include both.

3.  **Signature:**
    Using the HS256 algorithm, sign the encoded Header and Payload with your merchant key (`secretKey`).

**Example (Pseudocode - Python using PyJWT library):**
```python
import jwt
import time

# Your merchant information
mch_id = "YOUR_MCH_ID"
secret_key = "YOUR_SECRET_KEY" # This is your merchant key
user_id_from_your_system = "USER_XYZ789"

# ID obtained from Kiwi API
kiwi_payment_id = "P2024072510300012345678" # or kiwi_subscribe_id

# Set JWT expiration time (e.g., 6 hours later)
expiration_time = int(time.time()) + (6 * 60 * 60)

# Build Payload
payload = {
    "iss": mch_id,
    "aud": "kiwi",
    "userId": user_id_from_your_system,
    "exp": expiration_time
}

# Add specific ID based on flow type
is_one_time_payment = True # or False for subscription
if is_one_time_payment:
    payload["paymentId"] = kiwi_payment_id
else:
    payload["subscribeId"] = kiwi_subscribe_id # Use corresponding subscription ID

# Generate JWT
encoded_jwt = jwt.encode(payload, secret_key, algorithm="HS256")

# encoded_jwt is the token to be passed to the frontend
# print(encoded_jwt)
```

**Notes:**
- Make sure to use the official merchant key obtained from the Kiwi platform.
- Ensure the accuracy of `userId`, as this is key to authorizing users to access their orders/subscriptions.
- Many programming languages have mature JWT libraries that can simplify the generation process (such as Java's `jjwt`, Node.js's `jsonwebtoken`, etc.).

## `redirectURL` Usage Instructions

When calling the "Create Order" or "Create Subscription" interface, you can pass the `redirectURL` parameter. After the user completes the corresponding operation (such as successful payment, successful authorization) on the Kiwi payment/subscription UI, it will attempt to redirect the user's browser to this `redirectURL`.

**Two Main Scenarios:**

1.  **Redirect to Merchant-specified Page:**
    If you want users to return to a specific page in your application (such as order success page, member center) after completing operations on the Kiwi platform, you can set the complete URL of that page as the `redirectURL`.
    ```
    "redirectURL": "https://yourdomain.com/payment/thankyou?orderId=YOUR_ORDER_ID"
    ```
    The Kiwi platform will redirect the user's browser to `https://yourdomain.com/payment/thankyou?orderId=YOUR_ORDER_ID`.
    Your page can parse parameters in the URL (such as `orderId`) to display corresponding information.
    It's recommended to call your backend interface again in this `redirectURL` corresponding page to query the latest status of the order/subscription to ensure information accuracy, rather than relying solely on frontend redirection.

2.  **User Closes Payment/Subscription Window:**
    If you want users to directly close the Kiwi payment/subscription window or tab after completing operations (common in popup or new tab payment scenarios), you can set the `redirectURL` to a special value: `kiwi://close`.
    ```
    "redirectURL": "kiwi://close"
    ```
    When the Kiwi frontend UI detects this `redirectURL`, it will attempt to execute the `window.close()` operation.
    **Please note:** Browsers have security restrictions on the `window.close()` behavior. Typically, only windows opened by scripts can be closed by scripts. If the Kiwi payment page is not opened by your script (e.g., `window.open()`), `kiwi://close` may not close the window as expected, and users may need to close it manually.

**Not Providing `redirectURL`:**
If you don't provide a `redirectURL`, users will stay on the Kiwi platform's default result page after completing operations.

## Advanced Integration: Directly Display Receiving Address (No Redirection to Kiwi UI)

For merchants who want to more deeply customize the user experience, or don't want/need users to be redirected to the Kiwi payment page, you can choose to directly display the receiving address to users. This method is particularly suitable for API-driven processes or specific payment scenarios.

**Process Overview:**

1.  **Backend Creates Order/Subscription:**
    Your backend service normally calls the Kiwi platform's "Create Order" (`POST /api/v1/payments`) or "Create Subscription" (`POST /api/v1/subscribe/create`) interface.
    For detailed parameters, please refer to the corresponding sections above.

2.  **Get Receiving Address:**
    From the successful API response, extract the `data.deposit_address` field. This is the cryptocurrency address where users need to actually make payments.
    For one-time payments, this is the only payment address.
    For subscriptions, this is typically the address for initial payment or subsequent manual payments (if the subscription is not an automatic on-chain deduction type or the user chooses balance payment).

3.  **Merchant Frontend Displays Address:**
    Your application system (website, app, etc.) directly displays this `deposit_address` to users along with the expected payment amount and currency (usually USDT or other stablecoins, depending on your configuration and Kiwi platform support).
    You can also display a QR code to facilitate user scanning for payment.

    **Important Tips:**
    - **Clearly inform users of the payment amount and currency.**
    - **Notify users of the payment network/chain** (e.g., Ethereum (ERC20), TRON (TRC20), etc.), ensuring users don't transfer on the wrong chain resulting in loss of funds. The Kiwi platform typically assigns addresses for specific coins on specific chains.
    - **Set reasonable payment time reminders**, as orders themselves may have expiration times (`expireAt`).

4.  **Users Complete Payment Themselves:**
    Users transfer the corresponding amount of cryptocurrency to the displayed `deposit_address` through their own wallets.

5.  **Rely on Backend Notifications to Confirm Payment:**
    In this integration mode, **your system's order status updates completely depend on backend asynchronous notifications (Webhooks) sent by the Kiwi platform**. When the Kiwi platform detects that the corresponding address has received payment and confirms it, it will send notifications of successful payment or subscription status updates to your configured notification URL.
    - Regarding receiving and processing notifications, please strictly follow the instructions in [Application API Overview (REST_API_en.md)](REST_API_en.md#notification-mechanism), especially signature verification and idempotency handling.

**Comparison with Redirecting to Kiwi UI Method:**

-   **User Experience:** Merchants have complete control over the user interface, and users don't need to leave the merchant platform.
-   **Frontend Integration:** No need to handle JWT generation and frontend redirection logic, nor worry about the `token` query parameter.
-   **`redirectURL`:** In this mode, the `redirectURL` passed when creating orders in the backend will not be used for user interface redirection, because users don't access Kiwi's UI at all.
-   **Payment Status Updates:** Strictly dependent on backend Webhook notifications. Merchants need to implement their own user-side payment status polling or waiting prompts.
-   **Applicable Scenarios:** UI with high customization needs, API-driven automated processes, embedded payments where user redirection is not desired, etc.

**Notes:**
- Since users pay directly to addresses, merchants need to have good mechanisms to prompt users with accurate payment amounts, currencies, and networks, avoiding user operation errors.
- It's strongly recommended to clearly display the order's payment status in your user interface and update it promptly after receiving Kiwi backend notifications.

## Payment Notifications

Whether or not users return to your website via `redirectURL`, the Kiwi platform will notify the "notification URL" configured in your Kiwi merchant backend of payment or subscription status changes through a backend asynchronous notification mechanism.

Please rely on this asynchronous notification to update the order status in your system, as it is more reliable than frontend redirection.

### Notification Header Information

Notifications sent by the Kiwi platform will include the following HTTP headers:

-   **`Content-Type`**: `application/json;charset=utf-8`
-   **`X-KIWI-SIGN`**: Signature of the notification content, used to verify the authenticity of the notification.
-   **`X-KIWI-TIMESTAMP`**: Request timestamp (Unix seconds format, e.g., `1678234567`).
-   **`X-KIWI-NONCE`**: Random string, used to prevent replay attacks.

### Notification Signature Verification

Merchants should verify the authenticity of notifications through the following steps to ensure the notification comes from the Kiwi platform and the content has not been tampered with:

1.  **Prepare parameters participating in the signature:**
    *   `notifyURL`: The complete URL you configured in the Kiwi merchant backend to receive notifications.
    *   `requestBody`: The original JSON notification content (byte stream) obtained from the HTTP POST request.
    *   `timestamp`: The timestamp string obtained from the `X-KIWI-TIMESTAMP` request header.
    *   `nonce`: The random string obtained from the `X-KIWI-NONCE` request header.
    *   `secretKey`: Your merchant key obtained from the Kiwi platform.

2.  **Build Canonical String:**
    Concatenate the above parameters in the following format:
    `"POST" + notifyURL + "&" + Base64Encode(requestBody) + "&timestamp=" + timestamp + "&nonce=" + nonce + "&key=" + secretKey`

    *   `POST`: HTTP method, fixed as uppercase POST.
    *   `Base64Encode(requestBody)`: Base64 encode the original JSON notification content (byte stream).

3.  **Calculate Signature:**
    Calculate the HMAC-SHA256 hash value of the constructed canonical string using the merchant secret key, then convert the hash result to a hexadecimal string.

4.  **Compare Signatures:**
    Compare the calculated signature with the value in the `X-KIWI-SIGN` request header. If they match, signature verification passes.

**Example Code (Go):**
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

// verifyKiwiNotificationSignature verifies Kiwi notification signature
func verifyKiwiNotificationSignature(r *http.Request, secretKey string) (bool, error) {
    // Get necessary header information
    kiwiSign := r.Header.Get("X-KIWI-SIGN")
    kiwiTimestamp := r.Header.Get("X-KIWI-TIMESTAMP")
    kiwiNonce := r.Header.Get("X-KIWI-NONCE")

    if kiwiSign == "" || kiwiTimestamp == "" || kiwiNonce == "" {
        return false, fmt.Errorf("missing required Kiwi notification headers")
    }

    // Verify timestamp (e.g., valid within 5 minutes)
    timestampInt, err := strconv.ParseInt(kiwiTimestamp, 10, 64)
    if err != nil {
        return false, fmt.Errorf("invalid X-KIWI-TIMESTAMP format")
    }
    if time.Now().Unix()-timestampInt > 300 { // 5 minutes tolerance
        // return false, fmt.Errorf("X-KIWI-TIMESTAMP expired")
        // Note: For notifications, timestamp expiration checks can be more lenient or adjusted according to business needs, as retries may cause delays
    }

    // Read request body
    requestBody, err := ioutil.ReadAll(r.Body)
    if err != nil {
        return false, fmt.Errorf("failed to read request body: %w", err)
    }
    // Refill Body for subsequent processing (if needed)
    r.Body = ioutil.NopCloser(bytes.NewBuffer(requestBody))

    // Build canonical string
    // notifyURL is the URL you configured to receive notifications
    // In actual application, you need to know this URL
    notifyURL := r.URL.String() // This is usually the request path, you may need to concatenate with domain name etc. to get the complete URL
                                // Or directly use the URL string you configured in the Kiwi backend
    
    encodedBody := base64.StdEncoding.EncodeToString(requestBody)
    
    var parts []string
    parts = append(parts, "POST"+notifyURL)
    parts = append(parts, encodedBody) // body is base64 encoded
    parts = append(parts, "timestamp="+kiwiTimestamp)
    parts = append(parts, "nonce="+kiwiNonce)
    parts = append(parts, "key="+secretKey)

    // Connect parts with & symbol
    canonicalString := strings.Join(parts, "&")
    
    // Calculate HMAC-SHA256 hash
    mac := hmac.New(sha256.New, []byte(secretKey))
    mac.Write([]byte(canonicalString))
    calculatedSignature := hex.EncodeToString(mac.Sum(nil))
    
    // Compare signatures
    return calculatedSignature == kiwiSign, nil
}
```
**Note:** The `notifyURL` part in the above Go example needs to be replaced with the URL you configured in the Kiwi backend to receive notifications, according to your actual situation.

### Notification Processing Requirements

1.  **Idempotency Handling:** Due to network reasons or merchant server not responding in time, the Kiwi platform may send the same notification repeatedly. Merchant systems must be able to correctly handle duplicate notifications, ensuring business logic is executed only once. It's recommended to check the unique identifier in the notification (such as the combination of `payment_or_subscribe_id` and `event_type`) to determine if the notification has been processed.
2.  **Response Confirmation:** After receiving a notification, the merchant server must return HTTP status code `200` (OK) to confirm successful receipt. If any other status code is returned (including network timeout), the Kiwi platform will consider the notification failed and will retry according to the preset policy.
3.  **Retry Policy:** The Kiwi platform will make multiple retries for failed notifications. Retry intervals are roughly as follows:
    *   After first failure: 1 minute
    *   Then: 5 minutes, 10 minutes, 30 minutes
    *   Then: retry every 1 hour
    *   Maximum retry time: usually within 24 hours.
    Specific retry counts and intervals may be adjusted according to system configuration.
4.  **Timely Processing:** Merchants should process notifications and return responses promptly, avoiding the Kiwi platform misjudging as failure due to processing timeout and initiating retries. It's recommended to process time-consuming operations asynchronously.

### Event Types

The Kiwi platform sends notifications for different business events. The main event types (`event_type`) are as follows, and notification content will include core information related to that event:

**Important Notification Content Description:**
The JSON payload sent by the current version of the notification service mainly includes the following core identity identification fields. After receiving a notification, merchants should use these fields (especially `payment_or_subscribe_id` and `event_type`) to query detailed order/subscription/bill status and information through the API, rather than expecting the notification itself to contain all detailed data.

**Core Fields Included in Notifications:**
- `mch_id`: Merchant ID.
- `user_id`: User ID.
- `order_id`: Original merchant order number.
- `payment_or_subscribe_id`: Kiwi platform's payment ID, subscription ID, or bill ID, depending on the event context.
- `type`: Notification type, `ONE-TIME` or `SUBSCRIBE`.
- `event_type`: Current event type.

**1. One-Time Payment Related Notifications (`type: "ONE-TIME"`)**

-   **`PAYMENT_SUCCESS`**: Payment successful.
-   **`PAYMENT_FAILED`**: Payment failed.
-   **`PAYMENT_CLOSED`**: Order closed (e.g., manually closed).
-   **`PAYMENT_REFUNDED`**: Order refunded.

**2. Subscription Related Notifications (`type: "SUBSCRIBE"`)**

-   **`SUBSCRIBE_SUCCESS`**: Subscription creation/authorization successful.
    *   Trigger scenario: User confirms pending subscription through UI (calls `/pub/api/v1/user/subscribe/confirm/:id`) or user completes on-chain authorization for a subscription already in `SUBSCRIBED` status (calls `/pub/api/v1/user/personalSign/:id`).
    *   The `payment_or_subscribe_id` field is this subscription's `subscribe.ID`.
-   **`SUBSCRIBE_CANCELED`**: Subscription has been canceled.
    *   Trigger scenario: Merchant cancels subscription through API (`/api/v1/subscribe/cancel`) or user cancels subscription through UI (`/pub/api/v1/user/subscribe/cancel/:id`).
    *   The `payment_or_subscribe_id` field is this subscription's `subscribe.ID`.
-   **`SUBSCRIBE_CHARGE_SUCCESS`**: Subscription period charge successful.
    *   Trigger scenario: System successfully processes periodic bill through balance (automatically triggered after `/api/v1/subscribe/bill/create` if user balance is sufficient, or subsequently called by `ChargeBill` service) or on-chain payment subscription bill is confirmed successful (processed by `SubscribeCheckJob`).
    *   The `payment_or_subscribe_id` field is this successful `bill.ID`.
-   **`SUBSCRIBE_CHARGE_FAILED`**: Subscription period charge failed.
    *   Trigger scenario: On-chain payment subscription bill is confirmed failed or not confirmed for a long time (processed by `SubscribeCheckJob`).
    *   Note: Currently not all charge failure scenarios will trigger this notification (for example, when charging through balance, if the balance is insufficient and on-chain payment is not configured, or the `ChargeBill` service detects configuration issues before attempting on-chain payment, this notification may not be sent).
    *   The `payment_or_subscribe_id` field is this failed `bill.ID`.
-   **`SUBSCRIBE_STOPPED`**: Subscription stopped due to exceptional situation.
    *   Trigger scenario: Merchant stops subscription through API (`/api/v1/subscribe/stop`) (subscription status will change to `CANCELED`).
    *   The `payment_or_subscribe_id` field is this subscription's `subscribe.ID`.

### Notification Data Structure Examples

Below are JSON data structure examples for some common notification types. Please note again that actual notification content only includes core fields, and merchants should use these IDs to get detailed information through the API.

**Payment Success (PAYMENT_SUCCESS) Example:**
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
  "tx_hash": "0xabcdef1234567890..." // If it's on-chain payment
}
```

**Subscription Charge Success (SUBSCRIBE_CHARGE_SUCCESS) Example:**
```json
{
  "mch_id": "merchant123",
  "user_id": "user789",
  "order_id": "sub_order_xyz_2002", 
  "payment_or_subscribe_id": "B2024082611050055544332", // This is bill.ID
  "type": "SUBSCRIBE",
  "event_type": "SUBSCRIBE_CHARGE_SUCCESS"
}
```

**Subscription Charge Failed (SUBSCRIBE_CHARGE_FAILED) Example:**
```json
{
  "mch_id": "merchant123",
  "user_id": "user789",
  "order_id": "sub_order_xyz_2002", 
  "payment_or_subscribe_id": "B2024082611050055544333", // This is bill.ID
  "type": "SUBSCRIBE",
  "event_type": "SUBSCRIBE_CHARGE_FAILED"
}
```

**Subscription Stopped (SUBSCRIBE_STOPPED) Example:**
```json
{
  "mch_id": "merchant123",
  "user_id": "user789",
  "order_id": "sub_order_xyz_2002", 
  "payment_or_subscribe_id": "S2024072611000098765432", // This is subscribe.ID
  "type": "SUBSCRIBE",
  "event_type": "SUBSCRIBE_STOPPED"
}
```

---
If you have any questions, please contact Kiwi payment technical support.

### 3. Create Subscription Bill

Interface: POST /api/v1/subscribe/bill/create

Description: Create a collection bill for the specified subscription, typically called by the application in each collection cycle. The system will automatically calculate the next bill period and automatically cancel any previously unpaid bills.

Request Body:
```json
{
  "subscribeId": "Subscription ID - Required",
  "billDate": "Bill date - Optional, default is current time",
  "dueDate": "Bill due date - Optional, default is the last day of the month", 
  "fee": "Bill amount - Optional, default uses the subscription fee"
}
```

**Important Notes**:
- The system will automatically calculate the next bill period (the maximum period of all existing bills + 1)
- If there are bills with "pending payment" or "processing" status, the system will automatically set their status to "canceled" to ensure each subscription has only one active bill at a time
- This design allows merchants to create subscription bills at any time point when collection is needed, without having to worry about period management 