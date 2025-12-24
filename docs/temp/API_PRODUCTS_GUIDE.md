# Products API Developer Guide

**Last Updated**: December 2024
**Purpose**: Step-by-step guide for creating one-time purchase products via API
**Audience**: External developers integrating with Garu API

---

## Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
3. [Creating Your First Product](#creating-your-first-product)
4. [Getting the Payment Page Link](#getting-the-payment-page-link)
5. [Getting Transaction Details](#getting-transaction-details)
6. [API Reference](#api-reference)
7. [Code Examples](#code-examples)
8. [Webhooks](#webhooks)
9. [Best Practices](#best-practices)
10. [Troubleshooting](#troubleshooting)

---

## Overview

The Garu Products API allows you to programmatically create and manage products for one-time purchases. Products can accept payments via:

- **PIX** - Instant Brazilian payment method
- **Credit Card** - With installment options (1-12x)
- **Boleto** - Brazilian bank slip

### Base URL

```
Production: https://api.garu.com.br/api
Sandbox:    https://api.garu.com.br/api (use sk_test_ keys)
```

### API Endpoints

Two endpoint paths are available (both support API key authentication):

| Path | Description |
|------|-------------|
| `/api/products` | Main products API (recommended) |
| `/api/products` | V1 API (backward compatible) |

Both paths offer the same functionality. Examples in this guide use `/api/products`.

---

## Authentication

### Authentication Methods

The Garu Products API supports two authentication methods:

| Method | Header Format | Use Case |
|--------|---------------|----------|
| **API Key** | `Bearer sk_live_xxx` or `Bearer sk_test_xxx` | Server-to-server integrations |
| **JWT Token** | `Bearer eyJhbG...` | Dashboard, frontend apps |

All product endpoints (`/api/products`, `/api/subscription-prices`, `/api/portal/configuration`) accept both authentication methods.

### Getting Your API Key

1. Log in to your Garu dashboard at `https://app.garu.com.br`
2. Navigate to **Configurações** → **Desenvolvedores**
3. Click **Criar chave de API**
4. Select the environment:
   - `sk_test_` - For development and testing (no real charges)
   - `sk_live_` - For production (processes real payments)
5. **Save the key immediately** - It will only be shown once

### Using Your API Key

Include your API key in the `Authorization` header:

```bash
Authorization: Bearer sk_test_abc123xyz...
```

### Security Best Practices

- **Never expose** API keys in client-side code
- Store keys in environment variables
- Use `sk_test_` keys during development
- Rotate keys if compromised
- Use separate keys for different environments

---

## Creating Your First Product

### Step 1: Prepare Your Request

Gather the product information. Only the `name` is required - all other fields have sensible defaults:

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | **Yes** | - | Product name (displayed to customers) |
| `description` | string | No | `""` | Product description |
| `value` | number | No | `0` | Price in BRL (use 0 for subscription-only products) |
| `pix` | boolean | No | `true` | Enable PIX payments |
| `creditCard` | boolean | No | `true` | Enable credit card payments |
| `boleto` | boolean | No | `true` | Enable boleto payments |
| `installments` | number | No | `1` | Max installments (1-12) |

### Step 2: Make the API Request

**Minimal Example** (using all defaults):

```bash
curl -X POST https://api.garu.com.br/api/products \
  -H "Authorization: Bearer sk_test_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Meu Produto"
  }'
```

This creates a product with PIX, credit card, and boleto enabled, 1 installment, and price of R$ 0.00 (useful for subscription-only products).

**Full Example** (customizing all options):

```bash
curl -X POST https://api.garu.com.br/api/products \
  -H "Authorization: Bearer sk_test_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Curso de Marketing Digital",
    "description": "Aprenda marketing digital do zero ao avançado",
    "value": 297.00,
    "pix": true,
    "creditCard": true,
    "boleto": true,
    "installments": 12
  }'
```

### Step 3: Handle the Response

**Success Response (201 Created):**

```json
{
  "id": 123,
  "name": "Curso de Marketing Digital",
  "description": "Aprenda marketing digital do zero ao avançado",
  "value": 297.00,
  "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "pix": true,
  "creditCard": true,
  "boleto": true,
  "installments": [
    { "quantity": 1, "value": 297.00 },
    { "quantity": 2, "value": 153.45 },
    { "quantity": 3, "value": 104.94 },
    { "quantity": 4, "value": 80.69 },
    { "quantity": 5, "value": 66.15 },
    { "quantity": 6, "value": 56.52 },
    { "quantity": 7, "value": 49.68 },
    { "quantity": 8, "value": 44.55 },
    { "quantity": 9, "value": 40.59 },
    { "quantity": 10, "value": 37.42 },
    { "quantity": 11, "value": 34.85 },
    { "quantity": 12, "value": 32.72 }
  ],
  "isActive": true,
  "createdAt": "2024-12-24T10:30:00.000Z",
  "updatedAt": "2024-12-24T10:30:00.000Z"
}
```

### Step 4: Use the Product UUID

The `uuid` is used to build the public payment page URL:

```
https://garu.com.br/pay/{uuid}
```

Example: `https://garu.com.br/pay/a1b2c3d4-e5f6-7890-abcd-ef1234567890`

Share this URL with your customers to accept payments.

---

## Getting the Payment Page Link

After creating a product, you need to obtain the payment page link to share with your customers. There are three ways to get this link:

### Method 1: From the Create Product Response

When you create a product via the API, the response includes the `uuid` field. Use it to construct the payment URL:

```javascript
const response = await createProduct(data);
const paymentUrl = `https://garu.com.br/pay/${response.uuid}`;

console.log(paymentUrl);
// Output: https://garu.com.br/pay/a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

### Method 2: Retrieve an Existing Product

If you already have a product and need the payment link, fetch it by UUID:

```bash
# Get product details (public endpoint - no auth required)
curl -X GET https://api.garu.com.br/api/products/a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

**Response:**
```json
{
  "id": 123,
  "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "name": "Curso de Marketing Digital",
  "value": 297.00,
  ...
}
```

Then construct the URL: `https://garu.com.br/pay/{uuid}`

### Method 3: List All Products

Retrieve all your products and get their UUIDs:

```bash
curl -X GET https://api.garu.com.br/api/products \
  -H "Authorization: Bearer sk_test_your_api_key"
```

**Response:**
```json
{
  "data": [
    {
      "id": 123,
      "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "name": "Curso de Marketing Digital",
      "value": 297.00
    },
    {
      "id": 124,
      "uuid": "b2c3d4e5-f6a7-8901-bcde-f23456789012",
      "name": "E-book Python",
      "value": 47.00
    }
  ],
  "count": 2,
  "page": 1,
  "totalPages": 1
}
```

### Payment Page URL Format

| Environment | URL Format |
|-------------|------------|
| Production | `https://garu.com.br/pay/{uuid}` |
| Sandbox | `https://garu.com.br/pay/{uuid}` (same URL, test mode based on product) |

### Complete Code Example

```javascript
const GARU_BASE_URL = 'https://garu.com.br/pay';

// Function to create product and return payment link
async function createProductAndGetPaymentLink(productData) {
  const response = await fetch('https://api.garu.com.br/api/products', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.GARU_API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(productData)
  });

  if (!response.ok) {
    throw new Error('Failed to create product');
  }

  const product = await response.json();

  return {
    product,
    paymentLink: `${GARU_BASE_URL}/${product.uuid}`,
    // Store this UUID in your database for future reference
    uuid: product.uuid
  };
}

// Usage
const result = await createProductAndGetPaymentLink({
  name: 'Curso de Marketing Digital',
  description: 'Aprenda marketing digital',
  value: 297.00,
  pix: true,
  creditCard: true,
  boleto: true,
  installments: 12
});

console.log(`Product created: ${result.product.name}`);
console.log(`Payment link: ${result.paymentLink}`);
// Output:
// Product created: Curso de Marketing Digital
// Payment link: https://garu.com.br/pay/a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

### Using the Payment Link

Once you have the payment link, you can:

1. **Share directly with customers** via email, WhatsApp, or social media
2. **Embed in your website** as a button or link:
   ```html
   <a href="https://garu.com.br/pay/a1b2c3d4-e5f6-7890-abcd-ef1234567890"
      class="buy-button">
     Comprar Agora - R$ 297,00
   </a>
   ```
3. **Redirect from your application** after a user action:
   ```javascript
   function handleBuyClick() {
     window.location.href = `https://garu.com.br/pay/${productUuid}`;
   }
   ```
4. **Open in a modal/popup** for a seamless checkout experience:
   ```javascript
   function openCheckout(uuid) {
     window.open(
       `https://garu.com.br/pay/${uuid}`,
       'GaruCheckout',
       'width=500,height=700'
     );
   }
   ```

### Custom Return URL

Configure where customers are redirected after a successful purchase:

```json
{
  "name": "Curso de Marketing Digital",
  "value": 297.00,
  "pix": true,
  "creditCard": true,
  "boleto": true,
  "installments": 12,
  "returnUrl": "https://meusite.com.br/obrigado",
  "returnUrlButtonText": "Acessar Curso"
}
```

After payment, the customer will see a button with the text "Acessar Curso" that redirects to `https://meusite.com.br/obrigado`.

---

## Getting Transaction Details

After a customer completes a purchase, you can retrieve transaction details to verify payment status, access customer information, and manage refunds.

### Fetch a Single Transaction

Retrieve full details of a specific transaction by ID:

```bash
curl -X GET https://api.garu.com.br/api/transactions/12345 \
  -H "Authorization: Bearer sk_test_your_api_key"
```

**Response (200 OK):**

```json
{
  "id": 12345,
  "galaxPayId": 987654,
  "status": "captured",
  "value": 297.00,
  "valueSeller": 297.00,
  "paymentMethod": "creditcard",
  "installments": 3,
  "date": "2024-12-24T14:30:00.000Z",
  "deadline": "2024-12-24T14:30:00.000Z",
  "customer": {
    "id": 456,
    "name": "João Silva",
    "email": "joao@example.com",
    "document": "12345678901",
    "phone": "11999887766"
  },
  "product": {
    "id": 123,
    "name": "Curso de Marketing Digital",
    "value": 297.00,
    "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
  },
  "isRecurring": false,
  "subscriptionId": null,
  "billingCycle": null,
  "createdAt": "2024-12-24T14:30:00.000Z",
  "updatedAt": "2024-12-24T14:35:00.000Z"
}
```

### Transaction Status Values

| Status | Description |
|--------|-------------|
| `pendingBoleto` | Boleto generated, awaiting payment |
| `pendingPix` | PIX code generated, awaiting payment |
| `authorized` | Credit card authorized |
| `captured` | Credit card payment captured |
| `payedBoleto` | Boleto payment confirmed |
| `payedPix` | PIX payment confirmed |
| `denied` | Payment denied |
| `reversed` | Payment reversed/refunded |
| `cancel` | Transaction cancelled |
| `cancelByRecurrence` | Cancelled due to subscription cancellation |

### Check Transaction Status (Public Endpoint)

If you only need the payment status (not full details), use this public endpoint:

```bash
curl -X GET https://api.garu.com.br/api/transactions/status/987654
```

**Response:**
```
"captured"
```

**Note:** This endpoint uses the gateway's transaction ID (`galaxPayId`), not the internal transaction ID.

### Refund a Transaction

Issue a full or partial refund for a completed transaction:

```bash
curl -X POST https://api.garu.com.br/api/transactions/12345/refund \
  -H "Authorization: Bearer sk_test_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 100.00,
    "reason": "Customer requested refund"
  }'
```

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | number | No | Refund amount (defaults to full value if not specified) |
| `reason` | string | No | Reason for the refund |

**Response (200 OK):**

```json
{
  "id": 12345,
  "status": "reversed",
  "value": 297.00,
  "refundedAmount": 100.00,
  ...
}
```

### Cancel a Pending Transaction

Cancel a transaction before it's completed:

```bash
curl -X DELETE https://api.garu.com.br/api/transactions/12345 \
  -H "Authorization: Bearer sk_test_your_api_key"
```

**Response (200 OK):**

```json
true
```

### Code Example: Checking Payment Status

```javascript
async function getTransactionStatus(transactionId) {
  const response = await fetch(
    `https://api.garu.com.br/api/transactions/${transactionId}`,
    {
      headers: {
        'Authorization': `Bearer ${process.env.GARU_API_KEY}`
      }
    }
  );

  if (!response.ok) {
    throw new Error('Failed to fetch transaction');
  }

  const transaction = await response.json();

  // Check if payment is completed
  const paidStatuses = ['captured', 'payedBoleto', 'payedPix'];
  const isPaid = paidStatuses.includes(transaction.status);

  return {
    id: transaction.id,
    status: transaction.status,
    isPaid,
    value: transaction.value,
    paymentMethod: transaction.paymentMethod,
    customer: transaction.customer,
    product: transaction.product
  };
}

// Usage
const txStatus = await getTransactionStatus(12345);
if (txStatus.isPaid) {
  console.log(`Payment confirmed for ${txStatus.customer.name}`);
  // Grant access to product
}
```

---

## API Reference

### Create Product

```
POST /api/products
```

**Headers:**
| Header | Value |
|--------|-------|
| `Authorization` | `Bearer sk_test_xxx` or `Bearer sk_live_xxx` |
| `Content-Type` | `application/json` |

**Request Body:**

```typescript
{
  // Required field
  name: string;              // Product name

  // Optional fields with defaults
  description?: string;      // Product description (default: "")
  value?: number;            // Price in BRL (default: 0, use 0 for subscription products)
  pix?: boolean;             // Enable PIX (default: true)
  creditCard?: boolean;      // Enable credit card (default: true)
  boleto?: boolean;          // Enable boleto (default: true)
  installments?: number;     // Max installments 1-12 (default: 1)

  // Optional fields (no defaults)
  image?: string;            // Product image URL
  tags?: string[];           // Product tags/categories
  pixelFB?: string;          // Facebook Pixel ID
  returnUrl?: string;        // Redirect URL after purchase
  returnUrlButtonText?: string;  // Button text (max 100 chars)

  // Affiliate settings
  affiliationOrderAccept?: boolean;  // Accept affiliate orders
  displayOnAll?: boolean;            // Show to all affiliates
  leadBond?: boolean;                // Enable lead bonding
  comission?: string;                // Default commission (e.g., "10")
}
```

**Response (201):**

```typescript
{
  id: number;
  name: string;
  description: string;
  value: number;
  uuid: string;              // Unique identifier for payment page
  pix: boolean;
  creditCard: boolean;
  boleto: boolean;
  installments: Array<{ quantity: number; value: number }>;
  image?: string;
  tags?: string[];
  isActive: boolean;
  createdAt: string;
  updatedAt: string;
}
```

### Get Product by UUID

```
GET /api/products/{uuid}
```

**Note:** This endpoint is public and does not require authentication.

### Update Product

```
PATCH /api/products/{id}
```

**Headers:**
| Header | Value |
|--------|-------|
| `Authorization` | `Bearer sk_xxx` |
| `Content-Type` | `application/json` |

**Request Body:** Same as create (all fields optional)

### Delete Product (Soft Delete)

```
DELETE /api/products/{id}
```

Sets `isActive: false`. Product data is preserved for transaction history.

### List Products

```
GET /api/products
```

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | number | 1 | Page number |
| `limit` | number | 10 | Items per page |
| `tab` | string | "all" | Filter: "all", "mine", "affiliations" |

### Get Transaction by ID

```
GET /api/transactions/{id}
```

**Headers:**
| Header | Value |
|--------|-------|
| `Authorization` | `Bearer sk_xxx` |

**Response (200):**

```typescript
{
  id: number;
  galaxPayId: number;
  status: string;
  value: number;
  valueSeller: number;
  paymentMethod: string;
  installments: number;
  date: string;
  deadline: string;
  customer: {
    id: number;
    name: string;
    email: string;
    document: string;
    phone: string;
  };
  product: {
    id: number;
    name: string;
    value: number;
    uuid: string;
  };
  isRecurring: boolean;
  subscriptionId: number | null;
  billingCycle: number | null;
  createdAt: string;
  updatedAt: string;
}
```

### Get Transaction Status (Public)

```
GET /api/transactions/status/{galaxPayId}
```

Returns only the status string. No authentication required.

### Refund Transaction

```
POST /api/transactions/{id}/refund
```

**Headers:**
| Header | Value |
|--------|-------|
| `Authorization` | `Bearer sk_xxx` |
| `Content-Type` | `application/json` |

**Request Body:**

```typescript
{
  amount?: number;    // Refund amount (optional, defaults to full)
  reason?: string;    // Reason for refund (optional)
}
```

### Cancel Transaction

```
DELETE /api/transactions/{id}
```

**Headers:**
| Header | Value |
|--------|-------|
| `Authorization` | `Bearer sk_xxx` |

Returns `true` on success.

---

## Code Examples

### JavaScript / Node.js

```javascript
const createProduct = async () => {
  const response = await fetch('https://api.garu.com.br/api/products', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.GARU_API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      name: 'E-book Marketing Digital',
      description: 'Guia completo de marketing digital',
      value: 47.00,
      pix: true,
      creditCard: true,
      boleto: false,
      installments: 3,
      tags: ['ebook', 'marketing'],
      returnUrl: 'https://meusite.com.br/obrigado'
    })
  });

  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.message);
  }

  const product = await response.json();
  console.log(`Product created! Payment URL: https://garu.com.br/pay/${product.uuid}`);
  return product;
};
```

### Python

```python
import requests
import os

def create_product():
    url = "https://api.garu.com.br/api/products"
    headers = {
        "Authorization": f"Bearer {os.environ['GARU_API_KEY']}",
        "Content-Type": "application/json"
    }
    payload = {
        "name": "E-book Marketing Digital",
        "description": "Guia completo de marketing digital",
        "value": 47.00,
        "pix": True,
        "creditCard": True,
        "boleto": False,
        "installments": 3,
        "tags": ["ebook", "marketing"],
        "returnUrl": "https://meusite.com.br/obrigado"
    }

    response = requests.post(url, json=payload, headers=headers)
    response.raise_for_status()

    product = response.json()
    print(f"Product created! Payment URL: https://garu.com.br/pay/{product['uuid']}")
    return product
```

### PHP

```php
<?php

function createProduct() {
    $url = 'https://api.garu.com.br/api/products';
    $apiKey = getenv('GARU_API_KEY');

    $payload = [
        'name' => 'E-book Marketing Digital',
        'description' => 'Guia completo de marketing digital',
        'value' => 47.00,
        'pix' => true,
        'creditCard' => true,
        'boleto' => false,
        'installments' => 3,
        'tags' => ['ebook', 'marketing'],
        'returnUrl' => 'https://meusite.com.br/obrigado'
    ];

    $ch = curl_init($url);
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => [
            "Authorization: Bearer {$apiKey}",
            'Content-Type: application/json'
        ],
        CURLOPT_POSTFIELDS => json_encode($payload)
    ]);

    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);

    if ($httpCode !== 201) {
        throw new Exception("Error creating product: {$response}");
    }

    $product = json_decode($response, true);
    echo "Product created! Payment URL: https://garu.com.br/pay/{$product['uuid']}\n";
    return $product;
}
```

### cURL

```bash
# Create a product
curl -X POST https://api.garu.com.br/api/products \
  -H "Authorization: Bearer sk_test_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "E-book Marketing Digital",
    "description": "Guia completo de marketing digital",
    "value": 47.00,
    "pix": true,
    "creditCard": true,
    "boleto": false,
    "installments": 3,
    "tags": ["ebook", "marketing"],
    "returnUrl": "https://meusite.com.br/obrigado"
  }'

# Get product by UUID
curl -X GET https://api.garu.com.br/api/products/a1b2c3d4-e5f6-7890-abcd-ef1234567890

# Update product
curl -X PATCH https://api.garu.com.br/api/products/123 \
  -H "Authorization: Bearer sk_test_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "value": 57.00,
    "installments": 6
  }'

# Delete product (soft delete)
curl -X DELETE https://api.garu.com.br/api/products/123 \
  -H "Authorization: Bearer sk_test_your_api_key"
```

---

## Webhooks

### Overview

Receive real-time notifications when product events occur. Configure webhooks in the Garu dashboard under **Configurações** → **Webhooks**.

### Product Events

| Event | Description |
|-------|-------------|
| `product.created` | A new product was created |
| `product.updated` | A product was updated |
| `product.deleted` | A product was deactivated |

### Webhook Payload

```json
{
  "event": "product.created",
  "timestamp": "2024-12-24T10:30:00.000Z",
  "data": {
    "id": 123,
    "name": "Curso de Marketing Digital",
    "value": 297.00,
    "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "sellerId": 456
  }
}
```

### Verifying Webhook Signatures

Validate webhook authenticity using the signature header:

```javascript
const crypto = require('crypto');

function verifyWebhookSignature(payload, signature, secret) {
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(JSON.stringify(payload))
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expectedSignature)
  );
}

// In your webhook handler
app.post('/webhooks/garu', (req, res) => {
  const signature = req.headers['x-garu-signature'];

  if (!verifyWebhookSignature(req.body, signature, process.env.WEBHOOK_SECRET)) {
    return res.status(401).json({ error: 'Invalid signature' });
  }

  // Process webhook
  const { event, data } = req.body;
  console.log(`Received ${event} for product ${data.id}`);

  res.status(200).json({ received: true });
});
```

---

## Best Practices

### 1. Use Idempotency

For critical operations, implement idempotency on your side to prevent duplicate products:

```javascript
// Generate a unique key per creation request
const idempotencyKey = `product_${Date.now()}_${Math.random().toString(36)}`;

// Store the key and check before creating
if (await hasBeenProcessed(idempotencyKey)) {
  return existingProduct;
}

const product = await createProduct(data);
await markAsProcessed(idempotencyKey, product.id);
```

### 2. Handle Rate Limits

The API has a rate limit of 100 requests per minute. Handle 429 responses gracefully:

```javascript
const createProductWithRetry = async (data, retries = 3) => {
  for (let i = 0; i < retries; i++) {
    try {
      return await createProduct(data);
    } catch (error) {
      if (error.status === 429 && i < retries - 1) {
        const waitTime = Math.pow(2, i) * 1000; // Exponential backoff
        await new Promise(resolve => setTimeout(resolve, waitTime));
        continue;
      }
      throw error;
    }
  }
};
```

### 3. Validate Before Sending

Validate product data client-side before API calls. Since most fields are optional with defaults, validation is minimal:

```javascript
function validateProduct(data) {
  const errors = [];

  // Only name is required
  if (!data.name || data.name.trim() === '') {
    errors.push('Name is required');
  }

  // Optional validations (only if values are provided)
  if (data.value !== undefined && data.value < 0) {
    errors.push('Value must be at least 0');
  }

  if (data.installments !== undefined && (data.installments < 1 || data.installments > 12)) {
    errors.push('Installments must be between 1 and 12');
  }

  return errors;
}
```

> **Note**: Payment methods (`pix`, `creditCard`, `boleto`) default to `true`. If all are explicitly set to `false`, the product won't accept any payments.

### 4. Store Product UUIDs

Always store the product `uuid` in your database - you'll need it for payment page URLs.

### 5. Use Tags for Organization

Tags help organize products and enable filtering:

```json
{
  "name": "Curso Avançado de Python",
  "tags": ["curso", "programacao", "python", "avancado"]
}
```

---

## Troubleshooting

### Common Errors

#### 401 Unauthorized

```json
{
  "statusCode": 401,
  "message": "API key is required"
}
```

**Solution:** Check that your API key is included in the `Authorization` header with the `Bearer` prefix.

#### 400 Bad Request

```json
{
  "statusCode": 400,
  "message": ["value must be at least 5"]
}
```

**Solution:** Validate your request body against the required fields. Minimum price is R$ 5.00.

#### 403 Forbidden

```json
{
  "statusCode": 403,
  "message": "Insufficient permissions"
}
```

**Solution:** Your API key may not have product creation permissions. Check your key configuration in the dashboard.

#### 429 Too Many Requests

```json
{
  "statusCode": 429,
  "message": "Muitas tentativas. Por favor, tente novamente mais tarde."
}
```

**Solution:** You've exceeded the rate limit (100 req/min). Implement exponential backoff and retry logic.

### Testing Checklist

- [ ] API key is stored securely in environment variables
- [ ] Using `sk_test_` key for development
- [ ] Product name is provided (only required field)
- [ ] Webhook endpoint is configured and responding with 200
- [ ] Error handling is implemented for all API calls
- [ ] Product UUIDs are stored for payment page URLs

> **Tip**: With the simplified API, you can create a product with just the `name` field. All payment methods default to enabled, and installments default to 1.

### Support

If you encounter issues not covered here:

1. Check the [Swagger documentation](https://api.garu.com.br/api/swagger)
2. Contact support via the Garu dashboard
3. Email: suporte@garu.com.br

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2024-12-24 | Initial documentation |

---

**Next Steps:**
- [Transaction API Guide](./API_TRANSACTIONS_GUIDE.md) - Learn how to process payments
- [Webhook Configuration](./API_WEBHOOKS_GUIDE.md) - Set up real-time notifications
- [Subscription Guide](./API_SUBSCRIPTIONS_GUIDE.md) - Create recurring billing products
