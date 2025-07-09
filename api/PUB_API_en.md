# Public API Documentation (PUB_API_en.md)

## Overview

The Public API is a set of frontend-facing interfaces designed for users connecting via the Kiwi UI. These interfaces do not require formal authentication but may need JWT tokens to identify users or operations.

### Design Principles

1. These interfaces primarily serve the Kiwi frontend UI, but are also accessible to merchants who want to build their own user interfaces for certain operations.
2. All `/pub/api/*` endpoints support Cross-Origin Resource Sharing (CORS), allowing access from any domain's frontend.
3. APIs that require JWT validation will parse the user's identity from the token.
4. For security reasons, some interfaces may implement rate limiting to prevent abuse.

### Base URL

All public API endpoints are prefixed with: `/pub/api/v1`

## User Order Operations

### 1. Query User Order Status

**Endpoint:** `GET /pub/api/v1/user/payment/:id`

**Description:** Retrieve the status and details of a specific payment order. This interface requires a valid JWT with paymentId.

**JWT Requirements:**
- The JWT must include `paymentId` matching the `:id` in the URL path
- The JWT must include `userId` that matches the order's user ID

**Response Example:**
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "P202407251030001234567890",
    "mch_id": "merchant123",
    "user_id": "user456",
    "order_id": "order789",
    "total_fee": "99.99",
    "tax_fee": "9.99", 
    "created_at": "2024-07-25T10:30:00Z",
    "expire_at": "2024-07-25T11:30:00Z",
    "status": "PENDING_PAY",
    "order_type": "ONE_TIME",
    "deposit_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
    "memo": "Test order",
    "redirect_url": "https://merchant-site.com/order/success?orderId=order789",
    "tx_hash": null,
    "paid_at": null
  },
  "systemTime": 1721889000000
}
```

**Status Codes:**
- `PENDING_PAY`: Awaiting payment
- `PAID`: Payment completed successfully
- `EXPIRED`: Order has expired
- `CLOSED`: Order was manually closed
- `REFUNDED`: Payment was refunded

### 2. Query User Balance

**Endpoint:** `GET /pub/api/v1/user/balance`

**Description:** Check the user's available balance. Requires a valid JWT with userId.

**JWT Requirements:**
- The JWT must include `userId`

**Response Example:**
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "balance": "152.50",
    "currency": "USDT"
  },
  "systemTime": 1721889000000
}
```

### 3. Pay with Balance

**Endpoint:** `POST /pub/api/v1/user/balance/pay/:id`

**Description:** Attempt to pay for an order using the user's available balance. Requires a valid JWT with paymentId and userId.

**JWT Requirements:**
- The JWT must include `paymentId` matching the `:id` in the URL path
- The JWT must include `userId` that matches the order's user ID

**Request Body:** None required

**Response Example (Success):**
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "order_id": "order789",
    "payment_id": "P202407251030001234567890",
    "status": "PAID",
    "paid_at": "2024-07-25T10:45:00Z"
  },
  "systemTime": 1721889000000
}
```

**Response Example (Insufficient Balance):**
```json
{
  "code": 10001,
  "msg": "Insufficient balance",
  "data": {
    "required": "99.99",
    "available": "50.00"
  },
  "systemTime": 1721889000000
}
```

## Subscription Operations

### 1. Query Subscription Details

**Endpoint:** `GET /pub/api/v1/user/subscribe/:id`

**Description:** Retrieve the status and details of a subscription. Requires a valid JWT with subscribeId.

**JWT Requirements:**
- The JWT must include `subscribeId` matching the `:id` in the URL path
- The JWT must include `userId` that matches the subscription's user ID

**Response Example:**
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "id": "S202407251100001234567890",
    "mch_id": "merchant123",
    "user_id": "user456",
    "order_id": "subscription123",
    "fee": "19.99",
    "tax_fee": "1.99",
    "subscribe_total_count": 12,
    "subscribe_paid_count": 2,
    "subscribe_frequency": "Monthly",
    "created_at": "2024-07-25T11:00:00Z",
    "expire_at": null,
    "next_bill_date": "2024-09-25T11:00:00Z",
    "last_bill_date": "2024-08-25T11:00:00Z",
    "status": "SUBSCRIBED",
    "deposit_address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
    "personal_sign": true,
    "subscription_type": "SIGNATURE", 
    "memo": "Monthly membership"
  },
  "systemTime": 1721889000000
}
```

**Subscription Status Codes:**
- `PENDING_SUBSCRIBE`: Awaiting initial confirmation/payment
- `SUBSCRIBED`: Active subscription
- `CANCELED`: Subscription has been canceled
- `COMPLETED`: All subscription periods have been fulfilled
- `STOPPED`: Subscription was forcibly stopped (e.g., due to payment failures)
- `EXPIRED`: Initial subscription authorization expired without confirmation

### 2. Confirm Subscription

**Endpoint:** `POST /pub/api/v1/user/subscribe/confirm/:id`

**Description:** Confirm a pending subscription. For subscriptions that don't require initial payment, this changes the status from PENDING_SUBSCRIBE to SUBSCRIBED. Requires a valid JWT with subscribeId.

**JWT Requirements:**
- The JWT must include `subscribeId` matching the `:id` in the URL path
- The JWT must include `userId` that matches the subscription's user ID

**Request Body:** None required

**Response Example:**
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "subscribe_id": "S202407251100001234567890",
    "status": "SUBSCRIBED"
  },
  "systemTime": 1721889000000
}
```

### 3. Cancel Subscription

**Endpoint:** `POST /pub/api/v1/user/subscribe/cancel/:id`

**Description:** Cancel an active subscription. This operation is only allowed if the subscription is in the SUBSCRIBED state. Requires a valid JWT with subscribeId.

**JWT Requirements:**
- The JWT must include `subscribeId` matching the `:id` in the URL path
- The JWT must include `userId` that matches the subscription's user ID

**Request Body:** None required

**Response Example:**
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "subscribe_id": "S202407251100001234567890",
    "status": "CANCELED"
  },
  "systemTime": 1721889000000
}
```

### 4. Create Personal Sign

**Endpoint:** `POST /pub/api/v1/user/personalSign/:id`

**Description:** Generate and store a personal signature for a subscription, enabling automatic deductions. This is typically used for on-chain wallet authorization. Requires a valid JWT with subscribeId.

**JWT Requirements:**
- The JWT must include `subscribeId` matching the `:id` in the URL path
- The JWT must include `userId` that matches the subscription's user ID

**Request Body:**
```json
{
  "message": "Signed message content",
  "signature": "0x...",
  "account": "0x..."
}
```

**Response Example:**
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "subscribe_id": "S202407251100001234567890",
    "personal_sign": true
  },
  "systemTime": 1721889000000
}
```

## Wallet Connection

### 1. Connect Wallet

**Endpoint:** `POST /pub/api/v1/user/wallet/connect`

**Description:** Register a user's wallet address with their account. Requires a valid JWT with userId.

**JWT Requirements:**
- The JWT must include `userId`

**Request Body:**
```json
{
  "address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
  "chain": "ETH",
  "message": "I am connecting my wallet to Kiwi Payment Service",
  "signature": "0x..."
}
```

**Response Example:**
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
    "chain": "ETH",
    "connected_at": "2024-07-25T12:00:00Z"
  },
  "systemTime": 1721889000000
}
```

### 2. Query Connected Wallets

**Endpoint:** `GET /pub/api/v1/user/wallet/list`

**Description:** List all wallets connected to a user's account. Requires a valid JWT with userId.

**JWT Requirements:**
- The JWT must include `userId`

**Response Example:**
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "wallets": [
      {
        "address": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
        "chain": "ETH",
        "connected_at": "2024-07-25T12:00:00Z"
      },
      {
        "address": "0x8A0b1CF82AcA3C98CdD79b25F6D6a2E18AfD0f5E",
        "chain": "BSC",
        "connected_at": "2024-07-25T14:30:00Z"
      }
    ]
  },
  "systemTime": 1721889000000
}
```

## System Information

### 1. Get Supported Chains

**Endpoint:** `GET /pub/api/v1/system/chains`

**Description:** Retrieve a list of blockchain networks supported by the Kiwi payment system. No authentication required.

**Response Example:**
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "chains": [
      {
        "id": "ETH",
        "name": "Ethereum",
        "icon": "https://assets.kiwi.com/chains/eth.png",
        "enabled": true
      },
      {
        "id": "BSC",
        "name": "BNB Smart Chain",
        "icon": "https://assets.kiwi.com/chains/bsc.png",
        "enabled": true
      },
      {
        "id": "TRON",
        "name": "TRON",
        "icon": "https://assets.kiwi.com/chains/tron.png",
        "enabled": true
      }
    ]
  },
  "systemTime": 1721889000000
}
```


## Error Codes

| Code | Description |
|------|-------------|
| 1 | Success |
| 10000 | General error |
| 10001 | Insufficient balance |
| 10002 | Order expired |
| 10003 | Order already paid |
| 10004 | Order closed |
| 11000 | Subscription not found |
| 11001 | Subscription already confirmed |
| 11002 | Subscription expired |
| 11003 | Subscription already canceled |
| 11004 | Subscription cannot be canceled |
| 11005 | Personal sign failed |
| 20000 | JWT invalid or expired |
| 20001 | JWT missing required claims |
| 20002 | User ID mismatch |
| 20003 | Payment ID mismatch |
| 20004 | Subscription ID mismatch |
| 30000 | Wallet connection failed |
| 30001 | Invalid wallet signature |
| 30002 | Unsupported chain |
| 40000 | Rate limit exceeded |

## Common Use Cases

### Payment Flow

1. Merchant backend creates a payment order via the merchant API (`POST /api/v1/payments`)
2. Merchant redirects user to Kiwi payment UI with JWT
3. User views order details (`GET /pub/api/v1/user/payment/:id`)
4. User opts to pay with balance (`POST /pub/api/v1/user/balance/pay/:id`) or makes an on-chain payment
5. User is redirected to the merchant's redirectURL after payment completes

### Subscription Flow

1. Merchant backend creates a subscription via the merchant API (`POST /api/v1/subscribe/create`)
2. Merchant redirects user to Kiwi subscription UI with JWT
3. User views subscription details (`GET /pub/api/v1/user/subscribe/:id`)
4. User confirms subscription (`POST /pub/api/v1/user/subscribe/confirm/:id`)
5. For wallet-based subscriptions, user provides signature (`POST /pub/api/v1/user/personalSign/:id`)
6. Subscription becomes active
7. User can cancel subscription at any time (`POST /pub/api/v1/user/subscribe/cancel/:id`)

## Implementation Notes

1. **JWT Management**: Merchants should carefully manage JWT generation and ensure proper security measures, as these tokens grant access to user-specific operations.

2. **Error Handling**: Clients should handle HTTP status codes appropriately and parse the error codes from the response body for more detailed information.

3. **Rate Limiting**: Public API endpoints may implement rate limiting based on IP address or JWT claims to prevent abuse. Clients should implement appropriate retry mechanisms with exponential backoff.

4. **CORS**: All public API endpoints support Cross-Origin Resource Sharing, allowing them to be called directly from any frontend application.

5. **Mobile Applications**: These endpoints are also suitable for mobile applications, though additional security considerations may apply.

For detailed information about merchant integration, please refer to the [Merchant Integration Guide (MERCHANT_en.md)](docs/MERCHANT_en.md). 