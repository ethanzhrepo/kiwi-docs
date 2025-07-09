# Application API Overview (REST_API_en.md)

## Access Instructions

The application REST API is designed for backend server-to-server communication. Client applications can call these interfaces after authentication to complete operations such as creating orders, querying order status, and managing subscriptions.

### Base URL

The base URL for all API requests is: `https://api.domain.com/api/v1/`

### Authentication and Signature Mechanism

All requests require the following HTTP headers for authentication:

- `X-MCH-ID`: Your merchant ID, obtained during merchant registration.
- `X-Timestamp`: Request timestamp in RFC3339 format (e.g., `2023-01-01T12:00:00Z`).
- `X-Nonce`: Random string (16+ characters, hexadecimal only), used to prevent replay attacks.
- `X-Signature`: Request signature, calculated using the canonical HMAC-SHA256 signature algorithm described below.

#### Canonical Signature Generation

All API requests use a unified canonical signature method. The signature format is:

**API Request Signature Format:**
`Method + Path + "?" + SortedCanonicalQueryString + "&body=" + CanonicalRequestBody + "&timestamp=" + timestamp + "&nonce=" + nonce + "&key=" + secretKey`

**Components:**
- Method: HTTP method (GET, POST, etc.), uppercase
- Path: Request path (e.g., /api/v1/payments)
- SortedCanonicalQueryString: URL query parameters sorted alphabetically by key and URL-encoded, format: "key1=value1&key2=value2"
- CanonicalRequestBody: Base64-encoded request body with "&body=" prefix (omit entire body part if no request body)
- timestamp: Unix timestamp in seconds
- nonce: Random string (16+ characters, hexadecimal only)
- secretKey: Your merchant secret key

#### Signature Example (JavaScript)

```javascript
// Example in JavaScript
const crypto = require('crypto');

function generateCanonicalSignature(method, path, queryParams, requestBody, timestamp, nonce, secretKey) {
  // Normalize method to uppercase
  method = method.toUpperCase();
  
  // Build canonical query string (sorted by key)
  const queryKeys = Object.keys(queryParams).sort();
  const queryParts = [];
  
  for (const key of queryKeys) {
    const values = Array.isArray(queryParams[key]) ? queryParams[key] : [queryParams[key]];
    const sortedValues = values.sort();
    
    for (const value of sortedValues) {
      queryParts.push(`${encodeURIComponent(key)}=${encodeURIComponent(value)}`);
    }
  }
  const canonicalQueryString = queryParts.join('&');
  
  // Build parts of the canonical string
  const parts = [method, path];
  
  // Add query string if present
  if (canonicalQueryString) {
    parts.push(`?${canonicalQueryString}`);
  }
  
  // Add encoded body if present
  if (requestBody && requestBody.length > 0) {
    const encodedBody = Buffer.from(requestBody).toString('base64');
    parts.push(`&body=${encodedBody}`);
  }
  
  // Add timestamp, nonce, and key
  parts.push(
    `&timestamp=${timestamp}`,
    `&nonce=${nonce}`,
    `&key=${secretKey}`
  );
  
  // Join all parts
  const canonicalString = parts.join('');
  
  // Calculate HMAC-SHA256 hash
  const hash = crypto.createHmac('sha256', secretKey).update(canonicalString).digest('hex');
  
  return hash;
}

// Example usage
const method = 'POST';
const path = '/api/v1/payments';
const queryParams = { source: 'web' };
const requestBody = JSON.stringify({
  orderId: 'ORD123456',
  totalFee: '99.99',
  userId: 'USR123456'
});
const timestamp = Math.floor(Date.now() / 1000);
const nonce = crypto.randomBytes(16).toString('hex'); // Generate random hex string
const secretKey = 'your-merchant-secret-key';

const signature = generateCanonicalSignature(method, path, queryParams, requestBody, timestamp, nonce, secretKey);
console.log('X-Signature:', signature);

// Prepare request headers
const headers = {
  'Content-Type': 'application/json',
  'X-MCH-ID': 'your-merchant-id',
  'X-Timestamp': new Date(timestamp * 1000).toISOString(),
  'X-Nonce': nonce,
  'X-Signature': signature
};
```

#### Signature Example Walkthrough

HTTP Request:
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

1. Build canonical signature string:
   - Method: `POST`
   - Path: `/api/v1/payments`
   - SortedCanonicalQueryString: `source=web`
   - CanonicalRequestBody: `Base64({"amount":"100.00","userId":"user123"})`
     = `eyJhbW91bnQiOiIxMDAuMDAiLCJ1c2VySWQiOiJ1c2VyMTIzIn0=`
   - timestamp: `1672574400` (Unix timestamp for 2023-01-01T12:00:00Z)
   - nonce: `abc123`
   - secretKey: `your-secret-key`

2. Complete canonical string:
   `POST/api/v1/payments?source=web&body=eyJhbW91bnQiOiIxMDAuMDAiLCJ1c2VySWQiOiJ1c2VyMTIzIn0=&timestamp=1672574400&nonce=abc123&key=your-secret-key`

3. Calculate HMAC-SHA256:
   signature = HMAC-SHA256(canonical string, secretKey)

### Client Integration Guide

To simplify API integration, we recommend using our provided SDKs which have built-in canonical signature generation and verification.

#### Method 1: Using SDK (Recommended)

We provide SDKs in multiple languages with built-in canonical signature generation and verification. Example (Go):

```go
// Create API client
client := kiwi.NewClient("your-mch-id", "your-secret-key")

// Call API directly, SDK will automatically generate and add all authentication headers
response, err := client.CreateOrder(CreateOrderRequest{
    OrderID:  "order123",
    UserID:   "user456",
    TotalFee: "100.00",
})
```

#### Method 2: Manual Implementation

If you need to implement your own API client, ensure you follow the canonical signature method exactly as described above. Pay special attention to:

1. Query parameter sorting and URL encoding
2. Request body Base64 encoding with "&body=" prefix
3. Proper timestamp format handling (RFC3339 for headers, Unix seconds for signature)
4. Hexadecimal-only nonce generation (16+ characters)

#### Notification Signature (for webhook verification)

For webhook notifications, the system uses a similar but simplified signature format:

**Notification Signature Format:**
`"POST" + notifyURL + "&" + Base64(notificationBody) + "&timestamp=" + timestamp + "&nonce=" + nonce + "&key=" + secretKey`

**Components:**
- notifyURL: Your configured notification receiving URL
- notificationBody: Complete notification JSON content
- timestamp: Unix timestamp in seconds
- nonce: Random string
- secretKey: Your merchant secret key

**Notification Signature Example (JavaScript):**

```javascript
function generateNotificationSignature(notifyURL, notificationBody, timestamp, nonce, secretKey) {
  // Build canonical string parts
  const parts = [`POST${notifyURL}`];
  
  // Add encoded body if present
  if (notificationBody && notificationBody.length > 0) {
    const encodedBody = Buffer.from(notificationBody).toString('base64');
    parts.push(`&${encodedBody}`);
  }
  
  // Add timestamp, nonce, and key
  parts.push(
    `&timestamp=${timestamp}`,
    `&nonce=${nonce}`,
    `&key=${secretKey}`
  );
  
  // Join all parts
  const canonicalString = parts.join('');
  
  // Calculate HMAC-SHA256 hash
  const hash = crypto.createHmac('sha256', secretKey).update(canonicalString).digest('hex');
  
  return hash;
}

// Verify notification signature
function verifyNotificationSignature(notifyURL, requestBody, signature, timestamp, nonce, secretKey) {
  const expectedSignature = generateNotificationSignature(notifyURL, requestBody, timestamp, nonce, secretKey);
  return expectedSignature === signature;
}
```

### Important Notes

1. Signature calculation uses Unix second-level timestamp, while request headers use RFC3339 format timestamp
2. For request body signature, use the original JSON request body for Base64 encoding without any modifications or formatting
3. All API paths are provided as relative paths (excluding domain name or protocol)
4. Query parameters must be sorted alphabetically by key name
5. Each request's nonce must be unique, we recommend using UUID or other random values
6. Implement signature generation and verification according to the canonical method provided in this documentation, do not use deprecated old methods

### Request Body Size Limit
All API request bodies should not exceed 1MB. The server will reject requests exceeding this limit.

### API Rate Limiting
To ensure service stability, API interfaces have rate limits based on source IP address for specific path prefixes. Requests exceeding the limit will receive HTTP 429 (Too Many Requests) error. Default limits are:
- `/api/v1/payments` (payment-related interfaces): 1 request per second, allowing burst of 60 requests.
- `/api/v1/subscribe` (subscription-related interfaces): 1 request per second, allowing burst of 30 requests.
- `/pub/api/v1/` (public API interfaces): 20 requests per second, allowing burst of 100 requests.

Please control request frequency appropriately to avoid triggering limits.

### Request Format

For `GET` requests, parameters should be included in the URL query string.

For `POST` requests, the request body should be in JSON format with `Content-Type: application/json`.

### Response Format

All API responses are in JSON format with the following structure:

```json
{
  "code": 1,           // Response code, 1 for success, others for errors
  "msg": "success",    // Response message
  "data": {            // Response data, varies by endpoint
    // Specific data fields
  },
  "systemTime": 1689157536000  // System timestamp in milliseconds
}
```

## Common Response Codes

| Code | Description |
|------|-------------|
| 1    | Success |
| -1   | General error |
| 100  | Parameter error |
| 101  | Signature error |
| 102  | Request expired |
| 103  | Resource not found |
| 104  | Permission denied |
| 105  | Resource already exists |
| 106  | Resource state error |
| 107  | Balance insufficient |
| 108  | System error |

## API Endpoints

### 1. Query Order by Merchant Order ID

**Endpoint**: `GET /api/v1/payments/order?orderId=...`

**Description**: Query the status of a payment order using the merchant's original order ID.

**Parameters**:
- `orderId`: The merchant's original order ID (query parameter)

**Response Example**:
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "P2023071012345678",
    "mch_id": "your_mch_id",
    "user_id": "user_123",
    "order_id": "ord_456",
    "total_fee": "99.99",
    "tax_fee": "9.99",
    "created_at": "2023-07-10T12:34:56Z",
    "expire_at": "2023-07-10T13:34:56Z",
    "status": "PAID",
    "order_type": "ONE_TIME",
    "deposit_address": "0x123abc...",
    "memo": "Purchase description",
    "paid_at": "2023-07-10T12:45:30Z",
    "tx_hash": "0xabcdef123456...",
    "user_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F"
  },
  "systemTime": 1689157536000
}
```

**Order Status Values**:
- `PENDING_PAY`: Awaiting payment
- `PENDING_CONFIRM`: Payment received, awaiting confirmation
- `PAID`: Payment successful
- `CHARGE_FAILED`: Payment failed
- `EXPIRED`: Order expired
- `CLOSED`: Order closed
- `REFUNDED`: Payment refunded

### 2. Query Order Status by Payment ID

**Endpoint**: `GET /api/v1/payments/get?id=...`

**Description**: Query the status of a payment order using the payment ID returned when creating an order.

**Parameters**:
- `id`: The payment order ID (e.g., P2023071012345678) (query parameter)

**Response Example**:
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "P2023071012345678",
    "mch_id": "your_mch_id",
    "user_id": "user_123",
    "order_id": "ord_456",
    "total_fee": "99.99",
    "tax_fee": "9.99",
    "created_at": "2023-07-10T12:34:56Z",
    "expire_at": "2023-07-10T13:34:56Z",
    "status": "PAID",
    "order_type": "ONE_TIME",
    "deposit_address": "0x123abc...",
    "memo": "Purchase description",
    "paid_at": "2023-07-10T12:45:30Z",
    "tx_hash": "0xabcdef123456...",
    "user_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F"
  },
  "systemTime": 1689157536000
}
```

### 3. Refund Payment

**Endpoint**: `POST /api/v1/payments/refund?id=...`

**Description**: Process a full or partial refund for a paid order.

**Parameters**:
- `id`: The payment ID (query parameter)

**Request Body**:
```json
{
  "refundAmount": "50.00"
}
```

**Parameters**:
- `refundAmount`: Required, refund amount (must be positive and not exceed paid amount)

**Validation Requirements**:
- Payment status must be PAID
- For ONE_TIME orders: refund amount cannot exceed total order amount
- For SUBSCRIBE orders: refund amount cannot exceed actual paid total
- Each order can only be refunded once

**Response Example**:
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "message": "Refund processed successfully",
    "amount": "50.00"
  },
  "systemTime": 1689157536000
}
```

### 4. Create Subscription

**Endpoint**: `POST /api/v1/subscribe/create`

**Description**: Create a new subscription for recurring payments.

**Request Body**:
```json
{
  "orderId": "Your unique order ID",
  "userId": "User identifier in your system",
  "fee": "19.99",
  "taxFee": "1.99",
  "subscribeTotalCount": 12,
  "subscribeFrequency": "Monthly",
  "expireAt": "2023-08-10T00:00:00Z",
  "memo": "Monthly subscription",
  "redirectURL": "https://your-domain.com/subscribe/success?orderId=...",
  "logo": "https://your-domain.com/logo.png"
}
```

**Parameters**:
- `orderId`: Required, your system's unique order identifier
- `userId`: Required, user identifier in your system
- `fee`: Required, subscription fee per billing period in string format
- `taxFee`: Optional, tax portion of fee (included in fee) in string format
- `subscribeTotalCount`: Required, total number of billing periods
- `subscribeFrequency`: Optional, frequency (e.g., "Monthly", "Weekly", "Daily")
- `expireAt`: Optional, expiration time for first payment authorization
- `memo`: Optional, subscription description
- `redirectURL`: Optional, redirect URL after subscription confirmed
- `logo`: Optional, URL to your logo for display on payment page

**Response Example**:
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "S2023071012345678",
    "mch_id": "your_mch_id",
    "user_id": "user_123",
    "order_id": "ord_456",
    "fee": "19.99",
    "tax_fee": "1.99",
    "subscribe_total_count": 12,
    "subscribe_paid_count": 0,
    "subscribe_frequency": "Monthly",
    "created_at": "2023-07-10T12:34:56Z",
    "expire_at": "2023-08-10T00:00:00Z",
    "status": "PENDING_SUBSCRIBE",
    "deposit_address": "0x123abc...",
    "memo": "Monthly subscription",
    "redirect_url": "https://your-domain.com/subscribe/success?orderId=..."
  },
  "systemTime": 1689157536000
}
```

**Subscription Status Values**:
- `PENDING_SUBSCRIBE`: Awaiting user confirmation
- `SUBSCRIBED`: Active subscription
- `CANCELED`: Subscription canceled
- `COMPLETED`: All billing periods completed
- `STOPPED`: Subscription stopped due to issues
- `EXPIRED`: Initial authorization expired

### 5. Query Subscription Status

**Endpoint**: `GET /api/v1/subscribe/get`

**Description**: Query the status of a subscription using the subscription ID.

**Parameters**:
- `id`: The subscription ID (e.g., S2023071012345678) (query parameter)

**Response Example**:
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "S2023071012345678",
    "mch_id": "your_mch_id",
    "user_id": "user_123",
    "order_id": "ord_456",
    "fee": "19.99",
    "tax_fee": "1.99",
    "subscribe_total_count": 12,
    "subscribe_paid_count": 3,
    "subscribe_frequency": "Monthly",
    "created_at": "2023-07-10T12:34:56Z",
    "status": "SUBSCRIBED",
    "deposit_address": "0x123abc...",
    "memo": "Monthly subscription",
    "user_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F"
  },
  "systemTime": 1689157536000
}
```

### 6. Create Order

**Endpoint**: `POST /api/v1/payments`

**Description**: Create a new one-time payment order.

**Request Body**:
```json
{
  "orderId": "Your unique order ID",
  "userId": "User identifier in your system",
  "totalFee": "99.99",
  "taxFee": "9.99",
  "expireAt": "2023-07-10T13:34:56Z",
  "memo": "Purchase description",
  "redirectURL": "https://your-domain.com/payment/success?orderId=...",
  "logo": "https://your-domain.com/logo.png"
}
```

**Parameters**:
- `orderId`: Required, your system's unique order identifier
- `userId`: Required, user identifier in your system
- `totalFee`: Required, total order amount in string format
- `taxFee`: Optional, tax portion of total fee (included in totalFee) in string format
- `expireAt`: Optional, order expiration time
- `memo`: Optional, order description
- `redirectURL`: Optional, redirect URL after payment
- `logo`: Optional, URL to your logo for display on payment page

**Response Example**:
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "P2023071012345678",
    "mch_id": "your_mch_id",
    "user_id": "user_123",
    "order_id": "ord_456",
    "total_fee": "99.99",
    "tax_fee": "9.99",
    "created_at": "2023-07-10T12:34:56Z",
    "expire_at": "2023-07-10T13:34:56Z",
    "status": "PENDING_PAY",
    "order_type": "ONE_TIME",
    "deposit_address": "0x123abc...",
    "memo": "Purchase description",
    "redirect_url": "https://your-domain.com/payment/success?orderId=..."
  },
  "systemTime": 1689157536000
}
```

### 7. Cancel Subscription

**Endpoint**: `POST /api/v1/subscribe/cancel`

**Description**: Cancel an active subscription.

**Request Body**:
```json
{
  "subscribeId": "Subscription ID"
}
```

**Parameters**:
- `subscribeId`: Required, the subscription ID to cancel

**Response Example**:
```json
{
  "code": 1,
  "msg": "success",
  "data": "Subscription cancelled successfully",
  "systemTime": 1689157536000
}
```

### 8. Create Subscription Bill

**Endpoint**: `POST /api/v1/subscribe/bill/create`

**Description**: Create a bill for an existing subscription. The system will automatically calculate the next billing period, and automatically cancel any previous unpaid bills.

**Request Body**:
```json
{
  "subscribeId": "Subscription ID",
  "billDate": "2023-07-10T12:00:00Z",
  "dueDate": "2023-07-31T23:59:59Z",
  "fee": "19.99"
}
```

**Parameters**:
- `subscribeId`: Required, the subscription ID to bill
- `billDate`: Optional, the bill date (defaults to current time)
- `dueDate`: Optional, the due date (defaults to end of current month)
- `fee`: Optional, the bill amount (defaults to subscription fee)

**Response Example**:
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "B123456789012345678",
    "subscribe_id": "S2023071012345678",
    "mch_id": "your_mch_id",
    "user_id": "user_123",
    "fee": "19.99",
    "status": "PENDING",
    "created_at": "2023-07-10T12:00:00Z",
    "bill_date": "2023-07-10T12:00:00Z",
    "due_date": "2023-07-31T23:59:59Z",
    "billing_period": 1
  },
  "systemTime": 1689157536000
}
```

### 9. Get Subscription Bill by ID

**Endpoint**: `GET /api/v1/subscribe/bill/get`

**Description**: Get details of a subscription bill by its ID.

**Parameters**:
- `id`: Required, the bill ID (query parameter)

**Response Example**:
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "B123456789012345678",
    "subscribe_id": "S2023071012345678",
    "mch_id": "your_mch_id",
    "user_id": "user_123",
    "fee": "19.99",
    "status": "PENDING",
    "created_at": "2023-07-10T12:00:00Z",
    "bill_date": "2023-07-10T12:00:00Z",
    "due_date": "2023-07-31T23:59:59Z",
    "billing_period": 1
  },
  "systemTime": 1689157536000
}
```

### 10. Get Subscription Bill by Order ID and Period

**Endpoint**: `GET /api/v1/subscribe/bill/getByOrderIDAndPeriod`

**Description**: Get details of a subscription bill by order ID and billing period.

**Parameters**:
- `orderId`: Required, the merchant's order ID (query parameter)
- `period`: Required, the billing period number (query parameter)

**Response Example**:
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "B123456789012345678",
    "subscribe_id": "S2023071012345678",
    "mch_id": "your_mch_id",
    "user_id": "user_123",
    "fee": "19.99",
    "status": "PENDING",
    "created_at": "2023-07-10T12:00:00Z",
    "bill_date": "2023-07-10T12:00:00Z",
    "due_date": "2023-07-31T23:59:59Z",
    "billing_period": 1
  },
  "systemTime": 1689157536000
}
```

## Notification Mechanism

The Kiwi payment system will send HTTP POST notifications to your configured webhook URL when payment or subscription status changes occur. You should process these notifications to update your system's order status.

### Notification Security

Notifications include the following HTTP headers for verification:

- `X-KIWI-SIGN`: Signature for verification
- `X-KIWI-TIMESTAMP`: Timestamp when notification was sent (Unix seconds format)
- `X-KIWI-NONCE`: Random string to prevent replay attacks

To verify the notification's authenticity, use the notification signature method described in the Authentication section above. The verification process uses the canonical signature format specific to notifications:

```javascript
// Example notification signature verification
function verifyKiwiNotificationSignature(notifyURL, requestBody, signature, timestamp, nonce, secretKey) {
  // Generate expected signature using notification format
  const expectedSignature = generateNotificationSignature(notifyURL, requestBody, timestamp, nonce, secretKey);
  
  // Compare signatures
  return expectedSignature === signature;
}

// Usage in your webhook handler
app.post('/webhook', (req, res) => {
  const signature = req.headers['x-kiwi-sign'];
  const timestamp = req.headers['x-kiwi-timestamp'];
  const nonce = req.headers['x-kiwi-nonce'];
  
  // Your configured webhook URL
  const notifyURL = 'https://your-domain.com/webhook';
  
  // Read raw request body (you need to configure express to capture raw body)
  // Note: Use middleware like express.raw() or body-parser to get raw body bytes
  const requestBody = req.rawBody; // Raw body as received
  
  // Verify signature
  const isValid = verifyKiwiNotificationSignature(
    notifyURL,
    requestBody,
    signature,
    parseInt(timestamp),
    nonce,
    'your-secret-key'
  );
  
  if (!isValid) {
    return res.status(400).send('Invalid signature');
  }
  
  // Process notification...
  res.status(200).send('OK');
});
```

### Notification Types

The notification payload will include an `event_type` field indicating the event type:

**One-time payment events:**
- `PAYMENT_SUCCESS`: Payment successful
- `PAYMENT_FAILED`: Payment failed
- `PAYMENT_CLOSED`: Order closed
- `PAYMENT_REFUNDED`: Payment refunded

**Subscription events:**
- `SUBSCRIBE_SUCCESS`: Subscription activated
- `SUBSCRIBE_CANCELED`: Subscription canceled
- `SUBSCRIBE_CHARGE_SUCCESS`: Subscription period charge successful
- `SUBSCRIBE_CHARGE_FAILED`: Subscription period charge failed
- `SUBSCRIBE_STOPPED`: Subscription forcibly stopped

### Notification Example

```json
{
  "mch_id": "your_mch_id",
  "user_id": "user_123",
  "order_id": "ord_456",
  "payment_or_subscribe_id": "P2023071012345678",
  "type": "ONE-TIME",
  "event_type": "PAYMENT_SUCCESS",
  "total_fee": "99.99",
  "paid_at": "2023-07-10T12:45:30Z",
  "tx_hash": "0xabcdef123456..."
}
```

### Notification Response

Your server should respond with HTTP status code 200 to acknowledge receipt of the notification. If any other status code is returned, the notification system will retry sending the notification according to its retry policy.

### Notification Retry Policy

If your server doesn't respond with a 200 status code, the system will retry sending the notification with the following schedule:
- 1 minute after the first failure
- 5 minutes after the second failure
- 10 minutes after the third failure
- 30 minutes after the fourth failure
- Every hour thereafter up to 24 hours

### Notification Handling Best Practices

1. **Verify Signature**: Always verify the signature to ensure the notification is authentic.
2. **Idempotency**: Implement idempotent processing to handle potential duplicate notifications.
3. **Quick Response**: Respond quickly with a 200 status code, and process the notification asynchronously if needed.
4. **Logging**: Log all notifications for troubleshooting purposes.

## Rate Limits

To maintain system stability, API endpoints have the following rate limits:

- Create Order: 100 requests per minute per merchant
- Create Subscription: 50 requests per minute per merchant
- Query endpoints: 200 requests per minute per merchant

Exceeding these limits will result in HTTP 429 (Too Many Requests) responses.

## Error Handling

When errors occur, the API will return appropriate HTTP status codes along with a JSON response containing error details:

```json
{
  "code": 100,
  "msg": "Parameter error: totalFee is required",
  "data": null,
  "systemTime": 1689157536000
}
```

Common HTTP status codes:
- 400: Bad Request - Invalid parameters or request format
- 401: Unauthorized - Authentication failed
- 403: Forbidden - Insufficient permissions
- 404: Not Found - Resource not found
- 409: Conflict - Resource state conflict
- 429: Too Many Requests - Rate limit exceeded
- 500: Internal Server Error - Server error

## Testing

A sandbox environment is available for testing at `https://sandbox-api.domain.com/api/v1/`

In the sandbox environment:
- Use your test merchant credentials
- All payments are simulated
- Test credit cards and wallets are available

For additional technical support and access to the sandbox environment, please contact our technical support team.

## API Version and Changelog

Current API version: v1

**Changelog:**
- 2023-07-01: Initial release of API v1
- 2023-08-15: Added subscription management endpoints
- 2023-09-30: Added user balance query endpoint

For detailed information about the Public API for frontend applications, please refer to [Public API Documentation (PUB_API_en.md)](PUB_API_en.md). 