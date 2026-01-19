# Checkout Sessions API

The Checkout Sessions API allows you to create pre-configured payment links programmatically. Sessions support customer pre-fill, metadata, payment method restrictions, and webhook notifications.

## Overview

Checkout Sessions provide a Stripe-inspired workflow for collecting payments:

1. **Create a session** via API with product, customer data, and configuration
2. **Redirect customer** to the generated checkout URL
3. **Customer completes payment** on the hosted payment page
4. **Receive webhook** notification when payment is complete
5. **Customer is redirected** to your success URL

## Authentication

All API endpoints (except the public session endpoint) require API key authentication.

```bash
curl -X POST https://api.garu.com.br/api/checkout/sessions \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json"
```

## Base URL

```
https://api.garu.com.br/api
```

---

## Endpoints

### Create a Checkout Session

Creates a new checkout session and returns a payment URL.

```
POST /checkout/sessions
```

#### Headers

| Header | Type | Required | Description |
|--------|------|----------|-------------|
| `Authorization` | string | Yes | API key (`Bearer YOUR_API_KEY`) |
| `Content-Type` | string | Yes | `application/json` |
| `X-Idempotency-Key` | string | No | Unique key for request deduplication (24h window) |

#### Request Body

```json
{
  "product_id": 123,
  "price_id": "price_abc123",
  "customer": {
    "name": "Joao Silva",
    "email": "joao@email.com",
    "document": "12345678900",
    "phone": "11999999999",
    "zip_code": "01310100",
    "street": "Av. Paulista",
    "number": "1000",
    "complement": "Apt 123",
    "neighborhood": "Bela Vista",
    "city": "Sao Paulo",
    "state": "SP"
  },
  "payment_methods": ["pix", "creditCard"],
  "metadata": {
    "order_id": "12345",
    "campaign": "launch"
  },
  "success_url": "https://example.com/success?session_id={SESSION_ID}",
  "cancel_url": "https://example.com/cancel",
  "client_reference_id": "order_abc123",
  "affiliate_id": 456,
  "expires_at": "2025-01-20T12:00:00Z"
}
```

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `product_id` | integer | **Yes** | The ID of the product to sell |
| `price_id` | string | No | Subscription price ID (for recurring payments) |
| `customer` | object | No | Customer data for pre-filling the checkout form |
| `customer.name` | string | No | Customer's full name (max 255 chars) |
| `customer.email` | string | No | Customer's email address (max 255 chars) |
| `customer.document` | string | No | CPF or CNPJ (max 14 chars, numbers only) |
| `customer.phone` | string | No | Phone number (max 11 chars, numbers only) |
| `customer.zip_code` | string | No | ZIP code (max 8 chars, numbers only) |
| `customer.street` | string | No | Street address (max 255 chars) |
| `customer.number` | string | No | Address number (max 20 chars) |
| `customer.complement` | string | No | Address complement (max 255 chars) |
| `customer.neighborhood` | string | No | Neighborhood (max 100 chars) |
| `customer.city` | string | No | City (max 100 chars) |
| `customer.state` | string | No | State code (2 letters, e.g., "SP") |
| `payment_methods` | array | No | Restrict to specific payment methods: `pix`, `creditCard`, `boleto` |
| `metadata` | object | No | Key-value pairs for your use (max 50 keys, 500 chars per value) |
| `success_url` | string | No | Redirect URL after successful payment (max 500 chars) |
| `cancel_url` | string | No | Redirect URL if customer cancels (max 500 chars) |
| `client_reference_id` | string | No | Your internal reference ID (max 200 chars) |
| `affiliate_id` | integer | No | Affiliate ID for commission attribution |
| `expires_at` | string | No | Session expiration time (ISO 8601). Default: 24 hours from creation |

#### Response

```json
{
  "data": {
    "id": "cs_ABC123xyz",
    "status": "open",
    "url": "https://pay.garu.com.br/pay/session/abc123...",
    "product_id": 123,
    "price_id": "price_abc123",
    "customer_email": "joao@email.com",
    "customer_name": "Joao Silva",
    "metadata": {
      "order_id": "12345",
      "campaign": "launch"
    },
    "success_url": "https://example.com/success?session_id={SESSION_ID}",
    "cancel_url": "https://example.com/cancel",
    "client_reference_id": "order_abc123",
    "affiliate_id": 456,
    "transaction_id": null,
    "expires_at": "2025-01-20T12:00:00.000Z",
    "completed_at": null,
    "created_at": "2025-01-19T12:00:00.000Z"
  }
}
```

#### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique session identifier (use this for API calls) |
| `status` | string | Session status: `open`, `complete`, or `expired` |
| `url` | string | The checkout URL to redirect the customer to |
| `product_id` | integer | The product ID |
| `price_id` | string | Subscription price ID (if applicable) |
| `customer_email` | string | Pre-filled customer email |
| `customer_name` | string | Pre-filled customer name |
| `metadata` | object | Your metadata key-value pairs |
| `success_url` | string | Success redirect URL |
| `cancel_url` | string | Cancel redirect URL |
| `client_reference_id` | string | Your reference ID |
| `affiliate_id` | integer | Affiliate ID for attribution |
| `transaction_id` | integer | Transaction ID (populated when complete) |
| `expires_at` | string | Session expiration time (ISO 8601) |
| `completed_at` | string | Completion time (ISO 8601, null if not complete) |
| `created_at` | string | Creation time (ISO 8601) |

#### Status Codes

| Code | Description |
|------|-------------|
| `201` | Session created successfully |
| `400` | Validation error (invalid parameters) |
| `401` | Invalid or missing API key |
| `404` | Product not found |
| `429` | Rate limit exceeded |

---

### List Checkout Sessions

Retrieves a paginated list of checkout sessions.

```
GET /checkout/sessions
```

#### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | integer | 1 | Page number |
| `limit` | integer | 10 | Items per page (max 100) |
| `status` | string | - | Filter by status: `open`, `complete`, `expired` |

#### Example Request

```bash
curl -X GET "https://api.garu.com.br/api/checkout/sessions?page=1&limit=10&status=complete" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

#### Response

```json
{
  "data": [
    {
      "id": "cs_ABC123xyz",
      "status": "complete",
      "url": "https://pay.garu.com.br/pay/session/abc123...",
      "product_id": 123,
      "transaction_id": 789,
      "expires_at": "2025-01-20T12:00:00.000Z",
      "completed_at": "2025-01-19T14:30:00.000Z",
      "created_at": "2025-01-19T12:00:00.000Z"
    }
  ],
  "count": 1,
  "total_count": 150,
  "total_pages": 15
}
```

---

### Retrieve a Checkout Session

Retrieves details of a specific checkout session.

```
GET /checkout/sessions/{id}
```

#### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | string | The session ID (e.g., `cs_ABC123xyz`) |

#### Example Request

```bash
curl -X GET "https://api.garu.com.br/api/checkout/sessions/cs_ABC123xyz" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

#### Response

Returns the same structure as the create endpoint.

#### Status Codes

| Code | Description |
|------|-------------|
| `200` | Session found |
| `401` | Invalid or missing API key |
| `404` | Session not found |

---

### Expire a Checkout Session

Manually expires an open checkout session.

```
POST /checkout/sessions/{id}/expire
```

#### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | string | The session ID to expire |

#### Example Request

```bash
curl -X POST "https://api.garu.com.br/api/checkout/sessions/cs_ABC123xyz/expire" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

#### Response

Returns the updated session with `status: "expired"`.

#### Status Codes

| Code | Description |
|------|-------------|
| `200` | Session expired successfully |
| `401` | Invalid or missing API key |
| `404` | Session not found |
| `409` | Session cannot be expired (already complete or expired) |

---

## Idempotency

To safely retry requests without creating duplicate sessions, include an `X-Idempotency-Key` header:

```bash
curl -X POST https://api.garu.com.br/api/checkout/sessions \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "X-Idempotency-Key: unique-request-id-123" \
  -H "Content-Type: application/json" \
  -d '{"product_id": 123}'
```

**Key points:**
- Idempotency keys are valid for **24 hours**
- If you retry with the same key within 24h, you'll receive the original session
- Keys must be unique per API key (different sellers can use the same key)

---

## Webhooks

When a checkout session is completed, a webhook event is sent to your configured endpoint.

### Event: `checkout.session.completed`

```json
{
  "event": "checkout.session.completed",
  "data": {
    "id": "cs_ABC123xyz",
    "status": "complete",
    "product_id": 123,
    "transaction_id": 789,
    "customer_email": "joao@email.com",
    "metadata": {
      "order_id": "12345"
    },
    "client_reference_id": "order_abc123",
    "completed_at": "2025-01-19T14:30:00.000Z"
  },
  "created_at": "2025-01-19T14:30:01.000Z"
}
```

### Event: `checkout.session.expired`

```json
{
  "event": "checkout.session.expired",
  "data": {
    "id": "cs_ABC123xyz",
    "status": "expired",
    "product_id": 123,
    "metadata": {
      "order_id": "12345"
    },
    "client_reference_id": "order_abc123",
    "expires_at": "2025-01-20T12:00:00.000Z"
  },
  "created_at": "2025-01-20T12:00:01.000Z"
}
```

---

## Session Lifecycle

```
                    CHECKOUT SESSION LIFECYCLE

  CREATE -----> OPEN -----> COMPLETE
                 |              |
                 |              v
                 |        Transaction ID
                 |        is populated
                 |
                 v
              EXPIRED
              (after 24h or
               manual expire)
```

**Status Definitions:**

| Status | Description |
|--------|-------------|
| `open` | Session is active and awaiting payment |
| `complete` | Payment was successful, transaction created |
| `expired` | Session expired before payment (automatic after 24h or manual) |

---

## Rate Limits

| Endpoint | Limit |
|----------|-------|
| `POST /checkout/sessions` | 100 requests per minute |
| `GET /checkout/sessions` | 100 requests per minute |
| `GET /checkout/sessions/{id}` | 100 requests per minute |
| `POST /checkout/sessions/{id}/expire` | 100 requests per minute |

When rate limited, you'll receive a `429 Too Many Requests` response.

---

## Code Examples

### Node.js / JavaScript

```javascript
const createCheckoutSession = async () => {
  const response = await fetch('https://api.garu.com.br/api/checkout/sessions', {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_API_KEY',
      'Content-Type': 'application/json',
      'X-Idempotency-Key': `order_${Date.now()}`
    },
    body: JSON.stringify({
      product_id: 123,
      customer: {
        name: 'Joao Silva',
        email: 'joao@email.com'
      },
      success_url: 'https://yoursite.com/success?session_id={SESSION_ID}',
      cancel_url: 'https://yoursite.com/cancel',
      metadata: {
        order_id: 'order_123'
      }
    })
  });

  const { data } = await response.json();

  // Redirect customer to checkout
  window.location.href = data.url;
};
```

### Python

```python
import requests
import time

def create_checkout_session():
    response = requests.post(
        'https://api.garu.com.br/api/checkout/sessions',
        headers={
            'Authorization': 'Bearer YOUR_API_KEY',
            'Content-Type': 'application/json',
            'X-Idempotency-Key': f'order_{int(time.time())}'
        },
        json={
            'product_id': 123,
            'customer': {
                'name': 'Joao Silva',
                'email': 'joao@email.com'
            },
            'success_url': 'https://yoursite.com/success?session_id={SESSION_ID}',
            'cancel_url': 'https://yoursite.com/cancel',
            'metadata': {
                'order_id': 'order_123'
            }
        }
    )

    data = response.json()['data']
    return data['url']  # Redirect customer to this URL
```

### PHP

```php
<?php
function createCheckoutSession() {
    $ch = curl_init('https://api.garu.com.br/api/checkout/sessions');

    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_POST => true,
        CURLOPT_HTTPHEADER => [
            'Authorization: Bearer YOUR_API_KEY',
            'Content-Type: application/json',
            'X-Idempotency-Key: order_' . time()
        ],
        CURLOPT_POSTFIELDS => json_encode([
            'product_id' => 123,
            'customer' => [
                'name' => 'Joao Silva',
                'email' => 'joao@email.com'
            ],
            'success_url' => 'https://yoursite.com/success?session_id={SESSION_ID}',
            'cancel_url' => 'https://yoursite.com/cancel',
            'metadata' => [
                'order_id' => 'order_123'
            ]
        ])
    ]);

    $response = json_decode(curl_exec($ch), true);
    curl_close($ch);

    // Redirect customer to checkout URL
    header('Location: ' . $response['data']['url']);
    exit;
}
```

---

## Best Practices

### 1. Always Use Idempotency Keys

Prevent duplicate sessions by including a unique `X-Idempotency-Key` for each logical operation:

```bash
X-Idempotency-Key: order_12345_checkout_attempt_1
```

### 2. Store the Session ID

Save the `id` returned when creating a session to:
- Look up session status later
- Correlate with webhook events
- Debug payment issues

### 3. Use Metadata for Correlation

Store your internal IDs in the `metadata` or `client_reference_id` fields:

```json
{
  "metadata": {
    "order_id": "12345",
    "user_id": "user_abc"
  },
  "client_reference_id": "order_12345"
}
```

### 4. Handle Expiration Gracefully

Sessions expire after 24 hours by default. Implement proper handling:
- Check session status before assuming it's valid
- Create a new session if the previous one expired
- Show a friendly message if the customer returns to an expired session

### 5. Validate Webhooks

Always verify webhook signatures and check that:
- The session ID matches your records
- The `client_reference_id` corresponds to a valid order
- The status transition makes sense (e.g., `open` -> `complete`)

---

## Error Responses

All errors follow this format:

```json
{
  "statusCode": 400,
  "message": "Validation failed",
  "error": "Bad Request"
}
```

### Common Errors

| Status | Error | Description |
|--------|-------|-------------|
| `400` | `product_id is required` | Missing required product_id |
| `400` | `Invalid payment method` | Invalid value in payment_methods array |
| `401` | `Unauthorized` | Invalid or missing API key |
| `404` | `Product not found` | Product ID doesn't exist or doesn't belong to seller |
| `404` | `Checkout session not found` | Session ID doesn't exist |
| `409` | `Session cannot be expired` | Session is already complete or expired |
| `429` | `Too Many Requests` | Rate limit exceeded |

---

## Migration from Direct Links

If you're currently using direct product links (`/product-slug`), here's how to migrate:

**Before (Direct Link):**
```
https://pay.garu.com.br/meu-produto?email=joao@email.com
```

**After (Checkout Session):**
```javascript
// 1. Create session on your server
const session = await createCheckoutSession({
  product_id: 123,
  customer: { email: 'joao@email.com' }
});

// 2. Redirect to session URL
redirect(session.url);
// https://pay.garu.com.br/pay/session/abc123...
```

**Benefits:**
- Full control over customer data pre-fill
- Metadata for tracking and correlation
- Webhook notifications
- Idempotency support
- Session expiration control
