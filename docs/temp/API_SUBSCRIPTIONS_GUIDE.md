# Subscription Products API Guide

This guide explains how to create and manage subscription products through the Garu API. Subscriptions enable recurring billing for your customers with flexible billing intervals, trial periods, and automatic payment processing.

## Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
3. [Subscription Model](#subscription-model)
4. [Creating Subscription Products](#creating-subscription-products)
5. [Creating Subscription Prices](#creating-subscription-prices)
6. [Customer Checkout Flow](#customer-checkout-flow)
7. [Trial Periods](#trial-periods)
8. [Billing and Payment Processing](#billing-and-payment-processing)
9. [Payment Retry Logic](#payment-retry-logic)
10. [Cancellation](#cancellation)
11. [Getting Subscription Details](#getting-subscription-details)
12. [Customer Portal](#customer-portal)
13. [Webhooks](#webhooks)
14. [API Reference](#api-reference)
15. [Code Examples](#code-examples)
16. [Best Practices](#best-practices)
17. [Troubleshooting](#troubleshooting)

---

## Overview

Garu's subscription system allows you to:

- Create recurring billing products with flexible intervals (weekly, monthly, quarterly, annually)
- Offer trial periods before billing begins
- Process payments automatically via credit card
- Handle failed payments with automatic retry logic
- Allow customers to cancel with immediate or scheduled options
- Track subscription lifecycle through webhooks

### Key Concepts

| Term                   | Description                                                     |
| ---------------------- | --------------------------------------------------------------- |
| **Product**            | The item or service you're selling (e.g., "Premium Membership") |
| **Subscription Price** | A pricing plan for a product with billing interval and amount   |
| **Subscription**       | An active billing agreement between you and a customer          |
| **Trial Period**       | Free days before the first charge                               |
| **Billing Cycle**      | The recurring interval for charges                              |

---

## Authentication

All subscription endpoints support dual authentication:

### API Key Authentication (Recommended for Server-to-Server)

```bash
curl -X POST https://api.garu.com.br/api/subscription-prices \
  -H "Authorization: Bearer sk_live_your_api_key" \
  -H "Content-Type: application/json"
```

API keys follow the format:

- `sk_test_*` - Test environment (sandbox)
- `sk_live_*` - Production environment

### JWT Authentication (Dashboard/Frontend)

```bash
curl -X POST https://api.garu.com.br/api/subscription-prices \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -H "Content-Type: application/json"
```

---

## Subscription Model

### Billing Intervals

| Interval  | Value       | Description                        |
| --------- | ----------- | ---------------------------------- |
| Weekly    | `weekly`    | Charges every 7 days               |
| Monthly   | `monthly`   | Charges on the same day each month |
| Quarterly | `quarterly` | Charges every 3 months             |
| Annually  | `annually`  | Charges once per year              |

### Subscription Status Lifecycle

| Status                 | Description                                      |
| ---------------------- | ------------------------------------------------ |
| `pending`              | Subscription created, awaiting first payment     |
| `active`               | Subscription is active and billing normally      |
| `pending_cancellation` | Cancellation scheduled for end of billing period |
| `canceled`             | Subscription has been terminated                 |
| `failed`               | Payment failed after all retry attempts          |

---

## Creating Subscription Products

Before creating subscription prices, you need a product. Products can be either one-time purchase or subscription-based.

### Step 1: Create a Product

```bash
curl -X POST https://api.garu.com.br/api/products \
  -H "Authorization: Bearer sk_live_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Premium Membership",
    "description": "Access to all premium features",
    "productType": "subscription"
  }'
```

**Response:**

```json
{
  "id": 123,
  "uuid": "prod_abc123xyz",
  "name": "Premium Membership",
  "description": "Access to all premium features",
  "productType": "subscription",
  "isActive": true,
  "sellerId": 1,
  "createdAt": "2024-01-15T10:30:00.000Z"
}
```

### Product Types

| Type           | Description               |
| -------------- | ------------------------- |
| `one_time`     | Single purchase product   |
| `subscription` | Recurring billing product |

---

## Creating Subscription Prices

Subscription prices define how much and how often customers are billed.

### Step 2: Create a Subscription Price

```bash
curl -X POST https://api.garu.com.br/api/subscription-prices \
  -H "Authorization: Bearer sk_live_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "productId": 123,
    "name": "Monthly Plan",
    "amount": 4990,
    "billingInterval": "monthly",
    "trialDays": 7,
    "isActive": true
  }'
```

**Request Body:**

| Field             | Type    | Required | Description                                          |
| ----------------- | ------- | -------- | ---------------------------------------------------- |
| `productId`       | integer | Yes      | ID of the parent product                             |
| `name`            | string  | Yes      | Display name for this price/plan                     |
| `amount`          | integer | Yes      | Price in cents (4990 = R$ 49.90)                     |
| `billingInterval` | string  | Yes      | One of: `weekly`, `monthly`, `quarterly`, `annually` |
| `trialDays`       | integer | No       | Number of free trial days (0 = no trial)             |
| `isActive`        | boolean | No       | Whether this price is available (default: true)      |

**Response:**

```json
{
  "id": 456,
  "uuid": "price_def456uvw",
  "productId": 123,
  "name": "Monthly Plan",
  "amount": 4990,
  "billingInterval": "monthly",
  "trialDays": 7,
  "isActive": true,
  "sellerId": 1,
  "createdAt": "2024-01-15T10:35:00.000Z"
}
```

### Multiple Pricing Tiers

Create multiple prices for the same product to offer different plans:

```bash
# Basic Plan - Monthly
curl -X POST https://api.garu.com.br/api/subscription-prices \
  -H "Authorization: Bearer sk_live_your_api_key" \
  -d '{"productId": 123, "name": "Basic Monthly", "amount": 2990, "billingInterval": "monthly"}'

# Pro Plan - Monthly
curl -X POST https://api.garu.com.br/api/subscription-prices \
  -H "Authorization: Bearer sk_live_your_api_key" \
  -d '{"productId": 123, "name": "Pro Monthly", "amount": 4990, "billingInterval": "monthly"}'

# Pro Plan - Annual (with discount)
curl -X POST https://api.garu.com.br/api/subscription-prices \
  -H "Authorization: Bearer sk_live_your_api_key" \
  -d '{"productId": 123, "name": "Pro Annual", "amount": 47900, "billingInterval": "annually"}'
```

### Updating Subscription Prices

You can update existing subscription prices. Changes to pricing only affect **new subscriptions** - existing subscribers continue with their original terms.

#### Update Price Name and Description

```bash
curl -X PATCH https://api.garu.com.br/api/subscription-prices/456 \
  -H "Authorization: Bearer sk_live_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Pro Monthly (Best Value)",
    "description": "Full access to all pro features with priority support"
  }'
```

#### Update Pricing

```bash
curl -X PATCH https://api.garu.com.br/api/subscription-prices/456 \
  -H "Authorization: Bearer sk_live_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "unitAmount": 59.90
  }'
```

> **Important**: Changing `unitAmount` does NOT affect existing subscriptions. Current subscribers keep their original price until they cancel and resubscribe.

#### Update Trial Period

```bash
curl -X PATCH https://api.garu.com.br/api/subscription-prices/456 \
  -H "Authorization: Bearer sk_live_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "trialPeriodDays": 14
  }'
```

#### Deactivate a Price

Deactivating a price prevents new subscriptions but doesn't affect existing ones:

```bash
curl -X PATCH https://api.garu.com.br/api/subscription-prices/456 \
  -H "Authorization: Bearer sk_live_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "isActive": false
  }'
```

#### Add Custom Metadata

```bash
curl -X PATCH https://api.garu.com.br/api/subscription-prices/456 \
  -H "Authorization: Bearer sk_live_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "metadata": {
      "features": ["unlimited_api_calls", "priority_support", "custom_branding"],
      "tier": "pro"
    }
  }'
```

#### Updatable Fields

| Field                  | Type    | Description                                   |
| ---------------------- | ------- | --------------------------------------------- |
| `name`                 | string  | Plan display name                             |
| `description`          | string  | Plan description                              |
| `unitAmount`           | number  | Price in BRL (affects new subscriptions only) |
| `currency`             | string  | Currency code                                 |
| `billingInterval`      | string  | `weekly`, `monthly`, `quarterly`, `annually`  |
| `billingIntervalCount` | integer | Intervals between billings                    |
| `trialPeriodDays`      | integer | Free trial days                               |
| `isActive`             | boolean | Enable/disable for new subscriptions          |
| `metadata`             | object  | Custom key-value data                         |

---

## Customer Checkout Flow

### Step 3: Get the Payment Page Link

After creating a subscription price, customers can subscribe through the payment page.

**From Create Response:**

The payment page URL follows this pattern:

```
https://garu.com.br/pay/{product_uuid}?priceId={price_uuid}
```

**Retrieve Product with Prices:**

```bash
curl -X GET https://api.garu.com.br/api/products/123 \
  -H "Authorization: Bearer sk_live_your_api_key"
```

**Response:**

```json
{
  "id": 123,
  "uuid": "prod_abc123xyz",
  "name": "Premium Membership",
  "subscriptionPrices": [
    {
      "id": 456,
      "uuid": "price_def456uvw",
      "name": "Monthly Plan",
      "amount": 4990,
      "billingInterval": "monthly",
      "trialDays": 7
    }
  ]
}
```

**Payment Page URL:**

```
https://garu.com.br/pay/prod_abc123xyz?priceId=price_def456uvw
```

### Checkout Process

When a customer visits the payment page:

1. Customer enters their information (name, email, CPF/CNPJ, phone)
2. Customer enters credit card details
3. System validates the card and creates the subscription
4. If trial period exists, first charge is scheduled for after trial ends
5. Customer receives confirmation and subscription becomes active

---

## Trial Periods

Trial periods allow customers to try your subscription before being charged.

### How Trials Work

1. **Subscription Created**: Status becomes `active` immediately
2. **Trial Period**: Customer has full access, no charges
3. **Trial Ends**: First charge is processed automatically
4. **Billing Continues**: Regular billing cycle begins

### Setting Trial Days

```bash
curl -X POST https://api.garu.com.br/api/subscription-prices \
  -H "Authorization: Bearer sk_live_your_api_key" \
  -d '{
    "productId": 123,
    "name": "Premium with Trial",
    "amount": 4990,
    "billingInterval": "monthly",
    "trialDays": 14
  }'
```

### Trial Timeline Example

For a 14-day trial starting January 1st:

| Date     | Event                              |
| -------- | ---------------------------------- |
| Jan 1    | Subscription created, trial starts |
| Jan 1-14 | Trial period (no charges)          |
| Jan 15   | First charge of R$ 49.90           |
| Feb 15   | Second charge of R$ 49.90          |
| ...      | Continues monthly                  |

### Important Notes

- Trial days are calculated from subscription creation date
- Setting `trialDays: 0` means immediate billing
- Customers can cancel during trial without being charged
- Trial status is tracked in the subscription's `trialEndsAt` field

---

## Billing and Payment Processing

### How Billing Works

Garu handles subscription billing automatically, including:

- Credit card tokenization and secure storage
- Automatic recurring charges on the billing date
- Payment retry on failures
- Webhook notifications for payment events

### Billing Cycle

1. **Subscription Created**: First payment date calculated based on trial period
2. **Payment Due**: The system automatically charges the stored card
3. **Payment Success**: Subscription remains active, next charge scheduled
4. **Payment Failure**: Retry logic begins (see next section)

### Supported Payment Methods

| Method      | Support                  |
| ----------- | ------------------------ |
| Credit Card | Full support (recurring) |
| PIX         | One-time only            |
| Boleto      | One-time only            |

> **Note**: Only credit cards support automatic recurring billing. PIX and Boleto require manual payment each cycle.

### Billing Events

Each billing cycle generates subscription events:

```json
{
  "id": 789,
  "subscriptionId": 100,
  "eventType": "payment_success",
  "eventData": {
    "amount": 4990,
    "transactionId": "tx_abc123",
    "paidAt": "2024-02-15T10:00:00.000Z"
  },
  "createdAt": "2024-02-15T10:00:00.000Z"
}
```

---

## Payment Retry Logic

When a payment fails, the system automatically retries.

### Retry Strategy

| Attempt | Timing      | Action                          |
| ------- | ----------- | ------------------------------- |
| 1       | Immediately | First charge attempt            |
| 2       | +3 days     | Automatic retry                 |
| 3       | +3 days     | Automatic retry                 |
| 4       | +3 days     | Final attempt                   |
| Failed  | After 4th   | Subscription marked as `failed` |

### How It Works

1. **Initial Failure**: Payment declined, retry scheduled
2. **Customer Notified**: Email sent about payment issue
3. **Retry Attempts**: The system automatically retries up to 4 times
4. **Success**: Subscription continues normally
5. **Final Failure**: Subscription status changes to `failed`

### Payment Failure Reasons

| Reason               | Description                   |
| -------------------- | ----------------------------- |
| `insufficient_funds` | Card has insufficient balance |
| `card_expired`       | Card expiration date passed   |
| `card_declined`      | Bank declined the transaction |
| `invalid_card`       | Card number or CVV invalid    |
| `fraud_suspected`    | Fraud detection triggered     |

### Handling Failed Subscriptions

When a subscription fails after all retries:

1. Subscription status becomes `failed`
2. Webhook `subscription.payment_failed` is sent
3. Customer access should be revoked
4. Customer can reactivate by updating payment method

---

## Cancellation

Subscriptions can be canceled in two ways:

### Immediate Cancellation

Cancels the subscription right away. The customer loses access immediately.

```bash
curl -X POST https://api.garu.com.br/api/subscriptions/{subscriptionId}/cancel \
  -H "Authorization: Bearer sk_live_your_api_key" \
  -d '{
    "cancelImmediately": true
  }'
```

**Result:**

- Status changes to `canceled`
- Customer loses access immediately
- No further charges
- No prorated refunds (handle separately if needed)

### Scheduled Cancellation (End of Period)

Cancels at the end of the current billing period. Customer keeps access until then.

```bash
curl -X POST https://api.garu.com.br/api/subscriptions/{subscriptionId}/cancel \
  -H "Authorization: Bearer sk_live_your_api_key" \
  -d '{
    "cancelImmediately": false
  }'
```

**Result:**

- Status changes to `pending_cancellation`
- Customer keeps access until period ends
- `cancelAt` field shows when subscription will end
- No further charges after current period
- At period end, status changes to `canceled`

### Cancellation Timeline Example

For a monthly subscription billed on the 15th, canceled on January 20th:

**Immediate Cancellation:**
| Date | Event |
|------|-------|
| Jan 20 | Canceled, access revoked |

**Scheduled Cancellation:**
| Date | Event |
|------|-------|
| Jan 20 | Status: `pending_cancellation` |
| Jan 20 - Feb 15 | Customer has access |
| Feb 15 | Status: `canceled`, access revoked |

---

## Getting Subscription Details

### Fetch a Single Subscription

Retrieve complete details for a specific subscription:

```bash
curl -X GET https://api.garu.com.br/api/subscriptions/100 \
  -H "Authorization: Bearer sk_live_your_api_key"
```

**Response:**

```json
{
  "id": 100,
  "uuid": "sub_xyz789",
  "customerId": 50,
  "subscriptionPriceId": 456,
  "sellerId": 1,
  "status": "active",
  "currentPeriodStart": "2024-02-15T00:00:00.000Z",
  "currentPeriodEnd": "2024-03-15T00:00:00.000Z",
  "trialEndsAt": null,
  "cancelAt": null,
  "cancelledAt": null,
  "cancelAtPeriodEnd": false,
  "paymentMethodId": 789,
  "lastPaymentAt": "2024-02-15T10:00:00.000Z",
  "nextPaymentAt": "2024-03-15T10:00:00.000Z",
  "failedPaymentCount": 0,
  "createdAt": "2024-01-15T10:30:00.000Z",
  "updatedAt": "2024-02-15T10:00:00.000Z",
  "subscriptionPrice": {
    "id": 456,
    "uuid": "price_def456uvw",
    "name": "Monthly Plan",
    "unitAmount": 49.9,
    "billingInterval": "monthly",
    "product": {
      "id": 123,
      "name": "Premium Membership"
    }
  },
  "customer": {
    "id": 50,
    "name": "João Silva",
    "email": "joao@example.com"
  },
  "paymentMethod": {
    "id": 789,
    "cardLast4": "4242",
    "cardBrand": "visa"
  }
}
```

### List All Subscriptions

Retrieve all subscriptions with optional filters:

```bash
# Get all subscriptions
curl -X GET https://api.garu.com.br/api/subscriptions \
  -H "Authorization: Bearer sk_live_your_api_key"

# Filter by status
curl -X GET "https://api.garu.com.br/api/subscriptions?status=active" \
  -H "Authorization: Bearer sk_live_your_api_key"

# With pagination
curl -X GET "https://api.garu.com.br/api/subscriptions?page=1&limit=20" \
  -H "Authorization: Bearer sk_live_your_api_key"

# Search by customer name or email
curl -X GET "https://api.garu.com.br/api/subscriptions?search=joao@example.com" \
  -H "Authorization: Bearer sk_live_your_api_key"
```

**Response (Paginated):**

```json
{
  "data": [
    {
      "id": 100,
      "status": "active",
      "currentPeriodEnd": "2024-03-15T00:00:00.000Z",
      "customer": { "name": "João Silva", "email": "joao@example.com" },
      "subscriptionPrice": { "name": "Monthly Plan", "unitAmount": 49.9 }
    }
  ],
  "count": 1,
  "totalCount": 50,
  "totalPages": 3
}
```

### Query Parameters

| Parameter | Type    | Description                                                                         |
| --------- | ------- | ----------------------------------------------------------------------------------- |
| `page`    | integer | Page number (default: 1)                                                            |
| `limit`   | integer | Items per page (default: 20)                                                        |
| `status`  | string  | Filter by status: `active`, `pending`, `canceled`, `failed`, `pending_cancellation` |
| `search`  | string  | Search by customer name or email                                                    |

### Get Subscription Events (Billing History)

View all events for a subscription, including payments and status changes:

```bash
curl -X GET https://api.garu.com.br/api/subscriptions/100/events \
  -H "Authorization: Bearer sk_live_your_api_key"
```

**Response:**

```json
[
  {
    "id": 1001,
    "subscriptionId": 100,
    "eventType": "subscription.created",
    "eventData": {},
    "createdAt": "2024-01-15T10:30:00.000Z"
  },
  {
    "id": 1002,
    "subscriptionId": 100,
    "eventType": "payment_success",
    "eventData": {
      "amount": 4990,
      "transactionId": "tx_abc123"
    },
    "createdAt": "2024-02-15T10:00:00.000Z"
  }
]
```

---

## Customer Portal

The Customer Portal is a self-service interface that allows your subscribers to manage their subscriptions. You control access by generating secure session tokens for each customer.

### How the Portal Works

1. **You create a portal session** for a customer via API
2. **You receive a secure token** (valid for 1 hour)
3. **You redirect the customer** to the portal with the token
4. **Customer manages their subscription** based on your configuration

### Portal URL Format

```
https://garu.com.br/portal?token={session_token}
```

### Creating a Portal Session

Generate a secure session token to give a customer access to the portal:

```bash
curl -X POST https://api.garu.com.br/api/portal/sessions \
  -H "Authorization: Bearer sk_live_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": 50,
    "productId": 123,
    "returnUrl": "https://your-app.com/account"
  }'
```

**Request Body:**

| Field        | Type    | Required | Description                                          |
| ------------ | ------- | -------- | ---------------------------------------------------- |
| `customerId` | integer | Yes      | The customer's ID in Garu                            |
| `productId`  | integer | No       | Product ID for product-specific portal configuration |
| `returnUrl`  | string  | No       | URL to redirect after portal actions                 |

**Response:**

```json
{
  "sessionId": 789,
  "token": "a1b2c3d4e5f6789012345678901234567890abcdef1234567890abcdef123456",
  "expiresAt": "2024-01-15T11:30:00.000Z",
  "portalUrl": "/portal?token=a1b2c3d4e5f6..."
}
```

### Session Token Details

| Property       | Value                                             |
| -------------- | ------------------------------------------------- |
| **Format**     | 64-character hex string                           |
| **Expiration** | 1 hour from creation                              |
| **Security**   | Cryptographically random (crypto.randomBytes)     |
| **Single use** | Can be used multiple times within validity period |

### What Customers Can Do in the Portal

Based on your configuration, customers can:

| Action                | Configuration Flag         | Default |
| --------------------- | -------------------------- | ------- |
| Cancel subscription   | `allowCancelSubscription`  | `true`  |
| Update payment method | `allowUpdatePaymentMethod` | `true`  |
| Update billing info   | `allowUpdateBillingInfo`   | `true`  |
| View billing history  | `allowViewInvoices`        | `true`  |
| Pause subscription    | `allowCancelSubscription`  | `true`  |
| Resume subscription   | `allowCancelSubscription`  | `true`  |

### Configuring the Portal

#### Seller-Level Configuration (Default for All Products)

```bash
curl -X PUT https://api.garu.com.br/api/portal/configuration \
  -H "Authorization: Bearer sk_live_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "businessName": "Your Company Name",
    "logoUrl": "https://your-cdn.com/logo.png",
    "primaryColor": "#257264",
    "allowCancelSubscription": true,
    "allowUpdatePaymentMethod": true,
    "allowUpdateBillingInfo": true,
    "allowViewInvoices": true,
    "cancelAtPeriodEndOnly": true,
    "requireCancelReason": false,
    "sendCancellationEmail": true,
    "sendPaymentMethodUpdatedEmail": true
  }'
```

#### Product-Specific Configuration (Overrides Seller Defaults)

```bash
curl -X PUT https://api.garu.com.br/api/portal/configuration/product/123 \
  -H "Authorization: Bearer sk_live_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "allowCancelSubscription": false,
    "customWelcomeText": "Welcome to Premium Support Portal"
  }'
```

### Configuration Options

| Option                          | Type    | Description                                     |
| ------------------------------- | ------- | ----------------------------------------------- |
| `businessName`                  | string  | Your business name displayed in portal          |
| `logoUrl`                       | string  | URL to your logo                                |
| `primaryColor`                  | string  | Hex color for portal theme (default: `#257264`) |
| `allowCancelSubscription`       | boolean | Enable subscription cancellation                |
| `allowUpdatePaymentMethod`      | boolean | Enable payment method updates                   |
| `allowUpdateBillingInfo`        | boolean | Enable billing info updates                     |
| `allowViewInvoices`             | boolean | Enable billing history view                     |
| `cancelAtPeriodEndOnly`         | boolean | Force cancellations to schedule for period end  |
| `requireCancelReason`           | boolean | Require reason when canceling                   |
| `sendCancellationEmail`         | boolean | Send email on cancellation                      |
| `sendPaymentMethodUpdatedEmail` | boolean | Send email on payment method update             |
| `customWelcomeText`             | string  | Custom welcome message                          |
| `customSuccessMessage`          | string  | Custom success message                          |
| `customCancellationMessage`     | string  | Custom cancellation confirmation                |

### Portal API Endpoints (Token-Based)

All portal endpoints require the session token as a query parameter.

#### Get Portal Data

```bash
curl -X GET "https://api.garu.com.br/api/portal?token={session_token}"
```

Returns customer info, active subscriptions, and configuration.

#### Cancel Subscription

```bash
curl -X POST "https://api.garu.com.br/api/portal/subscriptions/{id}/cancel?token={session_token}"
```

#### Pause Subscription

```bash
curl -X POST "https://api.garu.com.br/api/portal/subscriptions/{id}/pause?token={session_token}"
```

#### Resume Subscription

```bash
curl -X POST "https://api.garu.com.br/api/portal/subscriptions/{id}/resume?token={session_token}"
```

#### Reactivate Cancelled Subscription

```bash
curl -X POST "https://api.garu.com.br/api/portal/subscriptions/{id}/reactivate?token={session_token}"
```

#### Update Payment Method

```bash
curl -X POST "https://api.garu.com.br/api/portal/subscriptions/{id}/update-payment-method?token={session_token}" \
  -H "Content-Type: application/json" \
  -d '{"paymentMethodId": 456}'
```

#### Add New Card and Update Subscription

```bash
curl -X POST "https://api.garu.com.br/api/portal/subscriptions/{id}/add-card-and-update?token={session_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "cardNumber": "4111111111111111",
    "holderName": "John Doe",
    "expiryMonth": 12,
    "expiryYear": 2025,
    "cvv": "123"
  }'
```

#### Get Billing History

```bash
curl -X GET "https://api.garu.com.br/api/portal/subscriptions/{id}/events?token={session_token}"
```

#### Update Customer Info

```bash
curl -X PUT "https://api.garu.com.br/api/portal/customer-info?token={session_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "John Doe",
    "email": "john@example.com",
    "phone": "11999998888"
  }'
```

### Integration Example

```javascript
class GaruPortal {
  constructor(apiKey) {
    this.apiKey = apiKey;
    this.baseUrl = "https://api.garu.com.br/api";
  }

  // Create a portal session for a customer
  async createPortalSession(customerId, options = {}) {
    const response = await fetch(`${this.baseUrl}/portal/sessions`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${this.apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        customerId,
        productId: options.productId,
        returnUrl: options.returnUrl || window.location.href,
      }),
    });

    return response.json();
  }

  // Redirect customer to portal
  async redirectToPortal(customerId, options = {}) {
    const session = await this.createPortalSession(customerId, options);
    window.location.href = `https://garu.com.br/portal?token=${session.token}`;
  }

  // Open portal in a new window
  async openPortalWindow(customerId, options = {}) {
    const session = await this.createPortalSession(customerId, options);
    window.open(
      `https://garu.com.br/portal?token=${session.token}`,
      "GaruPortal",
      "width=600,height=800"
    );
  }
}

// Usage
const portal = new GaruPortal("sk_live_your_api_key");

// Add "Manage Subscription" button to your app
document
  .getElementById("manage-subscription")
  .addEventListener("click", async () => {
    await portal.redirectToPortal(currentUser.garuCustomerId, {
      returnUrl: "https://your-app.com/account/subscriptions",
    });
  });
```

### Python Integration Example

```python
import requests

class GaruPortal:
    def __init__(self, api_key):
        self.api_key = api_key
        self.base_url = 'https://api.garu.com.br/api'
        self.headers = {
            'Authorization': f'Bearer {api_key}',
            'Content-Type': 'application/json'
        }

    def create_portal_session(self, customer_id, product_id=None, return_url=None):
        """Create a portal session and return the portal URL"""
        payload = {'customerId': customer_id}
        if product_id:
            payload['productId'] = product_id
        if return_url:
            payload['returnUrl'] = return_url

        response = requests.post(
            f'{self.base_url}/portal/sessions',
            headers=self.headers,
            json=payload
        )
        response.raise_for_status()
        session = response.json()

        return {
            'token': session['token'],
            'expires_at': session['expiresAt'],
            'portal_url': f'https://garu.com.br/portal?token={session["token"]}'
        }

    def configure_portal(self, config):
        """Update seller-level portal configuration"""
        response = requests.put(
            f'{self.base_url}/portal/configuration',
            headers=self.headers,
            json=config
        )
        response.raise_for_status()
        return response.json()


# Usage
portal = GaruPortal('sk_live_your_api_key')

# Configure portal once
portal.configure_portal({
    'businessName': 'My SaaS Company',
    'allowCancelSubscription': True,
    'cancelAtPeriodEndOnly': True
})

# Generate portal link for a customer
session = portal.create_portal_session(
    customer_id=50,
    return_url='https://my-app.com/account'
)

print(f"Send this link to your customer: {session['portal_url']}")
```

### Security Best Practices

1. **Never expose tokens in URLs you log** - Session tokens are sensitive
2. **Generate tokens on-demand** - Don't pre-generate and store tokens
3. **Use HTTPS returnUrl** - Always use secure URLs for redirects
4. **Validate customer ownership** - Ensure the customer belongs to your seller account
5. **Monitor portal activity** - Use webhooks to track customer actions

---

## Webhooks

Receive real-time notifications for subscription events.

### Configuring Webhooks

Set up webhook endpoints in your dashboard or via API:

```bash
curl -X POST https://api.garu.com.br/api/webhook-endpoints \
  -H "Authorization: Bearer sk_live_your_api_key" \
  -d '{
    "url": "https://your-app.com/webhooks/garu",
    "events": [
      "subscription.created",
      "subscription.activated",
      "subscription.payment_success",
      "subscription.payment_failed",
      "subscription.canceled",
      "subscription.trial_ending"
    ]
  }'
```

### Subscription Events

| Event                          | Description                  |
| ------------------------------ | ---------------------------- |
| `subscription.created`         | New subscription created     |
| `subscription.activated`       | Subscription became active   |
| `subscription.payment_success` | Recurring payment succeeded  |
| `subscription.payment_failed`  | Payment failed (may retry)   |
| `subscription.canceled`        | Subscription was canceled    |
| `subscription.trial_ending`    | Trial ends in 3 days         |
| `subscription.updated`         | Subscription details changed |

### Webhook Payload Example

```json
{
  "event": "subscription.payment_success",
  "timestamp": "2024-02-15T10:00:00.000Z",
  "data": {
    "subscription": {
      "id": 100,
      "uuid": "sub_xyz789",
      "status": "active",
      "customerId": 50,
      "subscriptionPriceId": 456,
      "currentPeriodStart": "2024-02-15T00:00:00.000Z",
      "currentPeriodEnd": "2024-03-15T00:00:00.000Z"
    },
    "payment": {
      "amount": 4990,
      "transactionId": "tx_abc123",
      "paidAt": "2024-02-15T10:00:00.000Z"
    }
  }
}
```

### Webhook Security

Verify webhook signatures to ensure authenticity:

```javascript
const crypto = require("crypto");

function verifyWebhook(payload, signature, secret) {
  const expectedSignature = crypto
    .createHmac("sha256", secret)
    .update(payload)
    .digest("hex");

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expectedSignature)
  );
}
```

---

## API Reference

### Subscription Prices

#### Create Subscription Price

```
POST /api/subscription-prices
```

| Field             | Type    | Required | Description                                  |
| ----------------- | ------- | -------- | -------------------------------------------- |
| `productId`       | integer | Yes      | Parent product ID                            |
| `name`            | string  | Yes      | Plan display name                            |
| `amount`          | integer | Yes      | Price in cents                               |
| `billingInterval` | string  | Yes      | `weekly`, `monthly`, `quarterly`, `annually` |
| `trialDays`       | integer | No       | Free trial days (default: 0)                 |
| `isActive`        | boolean | No       | Is available (default: true)                 |

#### List Subscription Prices

```
GET /api/subscription-prices
```

Returns all subscription prices for the authenticated seller.

#### Get Subscription Price

```
GET /api/subscription-prices/{id}
```

#### Update Subscription Price

```
PATCH /api/subscription-prices/{id}
```

| Field                  | Type    | Description                                                 |
| ---------------------- | ------- | ----------------------------------------------------------- |
| `name`                 | string  | Updated plan name                                           |
| `description`          | string  | Plan description                                            |
| `unitAmount`           | number  | Price in BRL (e.g., 49.90) - affects new subscriptions only |
| `currency`             | string  | Currency code (default: BRL)                                |
| `billingInterval`      | string  | `weekly`, `monthly`, `quarterly`, `annually`                |
| `billingIntervalCount` | integer | Number of intervals between billings                        |
| `trialPeriodDays`      | integer | Number of trial days                                        |
| `pricingModel`         | string  | Pricing model type                                          |
| `isActive`             | boolean | Enable/disable the plan                                     |
| `metadata`             | object  | Additional custom metadata                                  |

> **Note**: Updating `unitAmount` or `billingInterval` only affects new subscriptions. Existing subscriptions continue with their original terms.

#### Delete Subscription Price

```
DELETE /api/subscription-prices/{id}
```

> **Note**: Cannot delete prices with active subscriptions.

### Subscriptions

#### List Subscriptions

```
GET /api/subscriptions
```

Query parameters:
| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | string | Filter by status |
| `customerId` | integer | Filter by customer |
| `productId` | integer | Filter by product |

#### Get Subscription

```
GET /api/subscriptions/{id}
```

#### Cancel Subscription

```
POST /api/subscriptions/{id}/cancel
```

| Field               | Type    | Description                       |
| ------------------- | ------- | --------------------------------- |
| `cancelImmediately` | boolean | true = now, false = end of period |
| `reason`            | string  | Optional cancellation reason      |

#### Pause Subscription

```
POST /api/subscriptions/{id}/pause
```

Temporarily pauses billing for a subscription.

#### Resume Subscription

```
POST /api/subscriptions/{id}/resume
```

Resumes a paused subscription.

#### Get Subscription Events

```
GET /api/subscriptions/{id}/events
```

Returns the event history for a subscription (payments, status changes, etc.).

### Customer Portal

#### Create Portal Session

```
POST /api/portal/sessions
```

| Field        | Type    | Required | Description                                   |
| ------------ | ------- | -------- | --------------------------------------------- |
| `customerId` | integer | Yes      | Customer ID                                   |
| `productId`  | integer | No       | Product ID for product-specific configuration |
| `returnUrl`  | string  | No       | URL to redirect after portal actions          |

Returns session token valid for 1 hour.

#### Get Seller Portal Configuration

```
GET /api/portal/configuration
```

#### Update Seller Portal Configuration

```
PUT /api/portal/configuration
```

| Field                      | Type    | Description                       |
| -------------------------- | ------- | --------------------------------- |
| `businessName`             | string  | Business name displayed in portal |
| `logoUrl`                  | string  | Logo URL                          |
| `primaryColor`             | string  | Hex color for theme               |
| `allowCancelSubscription`  | boolean | Enable cancellation               |
| `allowUpdatePaymentMethod` | boolean | Enable payment method updates     |
| `allowUpdateBillingInfo`   | boolean | Enable billing info updates       |
| `allowViewInvoices`        | boolean | Enable billing history            |

#### Get Product Portal Configuration

```
GET /api/portal/configuration/product/{productId}
```

#### Update Product Portal Configuration

```
PUT /api/portal/configuration/product/{productId}
```

Overrides seller-level settings for a specific product.

#### Get Portal Data (Token-Based)

```
GET /api/portal?token={session_token}
```

Returns customer info, subscriptions, and configuration.

#### Cancel via Portal

```
POST /api/portal/subscriptions/{id}/cancel?token={session_token}
```

#### Pause via Portal

```
POST /api/portal/subscriptions/{id}/pause?token={session_token}
```

#### Resume via Portal

```
POST /api/portal/subscriptions/{id}/resume?token={session_token}
```

#### Reactivate via Portal

```
POST /api/portal/subscriptions/{id}/reactivate?token={session_token}
```

#### Update Payment Method via Portal

```
POST /api/portal/subscriptions/{id}/update-payment-method?token={session_token}
```

#### Add Card and Update via Portal

```
POST /api/portal/subscriptions/{id}/add-card-and-update?token={session_token}
```

#### Get Billing History via Portal

```
GET /api/portal/subscriptions/{id}/events?token={session_token}
```

#### Update Customer Info via Portal

```
PUT /api/portal/customer-info?token={session_token}
```

---

## Code Examples

### Node.js - Complete Integration

```javascript
const axios = require("axios");

class GaruSubscriptions {
  constructor(apiKey) {
    this.client = axios.create({
      baseURL: "https://api.garu.com.br/api",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
    });
  }

  // Create a subscription product
  async createProduct(name, description) {
    const response = await this.client.post("/products", {
      name,
      description,
      productType: "subscription",
    });
    return response.data;
  }

  // Create a subscription price/plan
  async createPrice(productId, options) {
    const response = await this.client.post("/subscription-prices", {
      productId,
      name: options.name,
      amount: options.amount,
      billingInterval: options.billingInterval,
      trialDays: options.trialDays || 0,
    });
    return response.data;
  }

  // Get checkout URL for a price
  getCheckoutUrl(productUuid, priceUuid) {
    return `https://garu.com.br/pay/${productUuid}?priceId=${priceUuid}`;
  }

  // List all subscriptions
  async listSubscriptions(filters = {}) {
    const response = await this.client.get("/subscriptions", {
      params: filters,
    });
    return response.data;
  }

  // Cancel a subscription
  async cancelSubscription(subscriptionId, immediately = false) {
    const response = await this.client.post(
      `/subscriptions/${subscriptionId}/cancel`,
      {
        cancelImmediately: immediately,
      }
    );
    return response.data;
  }
}

// Usage Example
async function main() {
  const garu = new GaruSubscriptions("sk_live_your_api_key");

  // 1. Create product
  const product = await garu.createProduct(
    "SaaS Platform",
    "Access to our SaaS platform"
  );
  console.log("Product created:", product.uuid);

  // 2. Create pricing plans
  const monthlyPlan = await garu.createPrice(product.id, {
    name: "Monthly",
    amount: 4990, // R$ 49.90
    billingInterval: "monthly",
    trialDays: 7,
  });

  const annualPlan = await garu.createPrice(product.id, {
    name: "Annual",
    amount: 47900, // R$ 479.00 (20% discount)
    billingInterval: "annually",
    trialDays: 14,
  });

  // 3. Get checkout URLs
  const monthlyUrl = garu.getCheckoutUrl(product.uuid, monthlyPlan.uuid);
  const annualUrl = garu.getCheckoutUrl(product.uuid, annualPlan.uuid);

  console.log("Monthly checkout:", monthlyUrl);
  console.log("Annual checkout:", annualUrl);
}

main().catch(console.error);
```

### Python - Complete Integration

```python
import requests

class GaruSubscriptions:
    def __init__(self, api_key):
        self.base_url = 'https://api.garu.com.br/api'
        self.headers = {
            'Authorization': f'Bearer {api_key}',
            'Content-Type': 'application/json'
        }

    def create_product(self, name, description):
        response = requests.post(
            f'{self.base_url}/products',
            headers=self.headers,
            json={
                'name': name,
                'description': description,
                'productType': 'subscription'
            }
        )
        response.raise_for_status()
        return response.json()

    def create_price(self, product_id, name, amount, billing_interval, trial_days=0):
        response = requests.post(
            f'{self.base_url}/subscription-prices',
            headers=self.headers,
            json={
                'productId': product_id,
                'name': name,
                'amount': amount,
                'billingInterval': billing_interval,
                'trialDays': trial_days
            }
        )
        response.raise_for_status()
        return response.json()

    def get_checkout_url(self, product_uuid, price_uuid):
        return f'https://garu.com.br/pay/{product_uuid}?priceId={price_uuid}'

    def list_subscriptions(self, status=None, customer_id=None):
        params = {}
        if status:
            params['status'] = status
        if customer_id:
            params['customerId'] = customer_id

        response = requests.get(
            f'{self.base_url}/subscriptions',
            headers=self.headers,
            params=params
        )
        response.raise_for_status()
        return response.json()

    def cancel_subscription(self, subscription_id, immediately=False):
        response = requests.post(
            f'{self.base_url}/subscriptions/{subscription_id}/cancel',
            headers=self.headers,
            json={'cancelImmediately': immediately}
        )
        response.raise_for_status()
        return response.json()


# Usage Example
if __name__ == '__main__':
    garu = GaruSubscriptions('sk_live_your_api_key')

    # Create product
    product = garu.create_product(
        'Premium Access',
        'Full access to premium features'
    )
    print(f"Product created: {product['uuid']}")

    # Create monthly plan with 7-day trial
    monthly = garu.create_price(
        product_id=product['id'],
        name='Monthly Plan',
        amount=2990,  # R$ 29.90
        billing_interval='monthly',
        trial_days=7
    )

    # Get checkout URL
    checkout_url = garu.get_checkout_url(product['uuid'], monthly['uuid'])
    print(f"Checkout URL: {checkout_url}")

    # List active subscriptions
    active_subs = garu.list_subscriptions(status='active')
    print(f"Active subscriptions: {len(active_subs)}")
```

### Webhook Handler (Express.js)

```javascript
const express = require("express");
const crypto = require("crypto");

const app = express();
app.use(express.raw({ type: "application/json" }));

const WEBHOOK_SECRET = process.env.GARU_WEBHOOK_SECRET;

app.post("/webhooks/garu", (req, res) => {
  const signature = req.headers["x-garu-signature"];
  const payload = req.body.toString();

  // Verify signature
  const expectedSig = crypto
    .createHmac("sha256", WEBHOOK_SECRET)
    .update(payload)
    .digest("hex");

  if (signature !== expectedSig) {
    return res.status(401).json({ error: "Invalid signature" });
  }

  const event = JSON.parse(payload);

  switch (event.event) {
    case "subscription.created":
      console.log("New subscription:", event.data.subscription.id);
      // Provision access for customer
      break;

    case "subscription.payment_success":
      console.log("Payment received:", event.data.payment.amount);
      // Update customer billing record
      break;

    case "subscription.payment_failed":
      console.log("Payment failed for:", event.data.subscription.id);
      // Send dunning email to customer
      break;

    case "subscription.canceled":
      console.log("Subscription canceled:", event.data.subscription.id);
      // Revoke customer access
      break;

    case "subscription.trial_ending":
      console.log("Trial ending soon:", event.data.subscription.id);
      // Send reminder email
      break;

    default:
      console.log("Unhandled event:", event.event);
  }

  res.json({ received: true });
});

app.listen(3000, () => console.log("Webhook server running on port 3000"));
```

---

## Best Practices

### Product Setup

1. **Clear naming**: Use descriptive names for products and plans
2. **Consistent pricing**: Keep pricing logical across intervals (annual should be discounted)
3. **Strategic trials**: Offer trials long enough for customers to see value

### Billing

1. **Test first**: Use `sk_test_*` keys to test the full subscription flow
2. **Handle failures gracefully**: Implement dunning emails for failed payments
3. **Monitor retries**: Track payment retry rates and adjust if needed

### Customer Experience

1. **Transparent pricing**: Show billing intervals and amounts clearly
2. **Easy cancellation**: Don't hide the cancel button
3. **Cancellation feedback**: Ask why customers are leaving

### Technical

1. **Idempotency**: Handle duplicate webhooks gracefully
2. **Webhook reliability**: Implement retry logic for webhook processing
3. **Status sync**: Periodically sync subscription status with your database

---

## Troubleshooting

### Common Issues

#### "Subscription price not found"

**Cause**: Invalid `priceId` or price belongs to different seller.

**Solution**: Verify the price ID and ensure you're using the correct API key.

#### "Payment method required"

**Cause**: Customer hasn't added a credit card.

**Solution**: Ensure customers complete the checkout flow with valid card details.

#### "Card declined"

**Cause**: Customer's card was rejected by the bank.

**Solution**: Ask customer to try a different card or contact their bank.

#### "Subscription already canceled"

**Cause**: Attempting to cancel an already canceled subscription.

**Solution**: Check subscription status before attempting cancellation.

### Debugging Tips

1. **Check subscription status**: Use `GET /api/subscriptions/{id}` to see current status
2. **Review events**: Check subscription events for payment history
3. **Webhook logs**: Review webhook delivery logs in the dashboard
4. **Test environment**: Use test API keys to simulate scenarios

### Support

For additional help:

- Dashboard: https://app.garu.com.br
- Documentation: https://docs.garu.com.br
- Email: suporte@garu.com.br

---

## Appendix: Complete Field Reference

### Subscription Object

```json
{
  "id": 100,
  "uuid": "sub_xyz789",
  "customerId": 50,
  "subscriptionPriceId": 456,
  "sellerId": 1,
  "status": "active",
  "currentPeriodStart": "2024-02-15T00:00:00.000Z",
  "currentPeriodEnd": "2024-03-15T00:00:00.000Z",
  "trialEndsAt": null,
  "cancelAt": null,
  "canceledAt": null,
  "paymentMethod": "credit_card",
  "lastPaymentAt": "2024-02-15T10:00:00.000Z",
  "nextPaymentAt": "2024-03-15T10:00:00.000Z",
  "failedPaymentCount": 0,
  "createdAt": "2024-01-15T10:30:00.000Z",
  "updatedAt": "2024-02-15T10:00:00.000Z"
}
```

### Subscription Price Object

```json
{
  "id": 456,
  "uuid": "price_def456uvw",
  "productId": 123,
  "sellerId": 1,
  "name": "Monthly Plan",
  "amount": 4990,
  "billingInterval": "monthly",
  "trialDays": 7,
  "isActive": true,
  "createdAt": "2024-01-15T10:35:00.000Z",
  "updatedAt": "2024-01-15T10:35:00.000Z"
}
```

### Subscription Event Object

```json
{
  "id": 789,
  "subscriptionId": 100,
  "eventType": "payment_success",
  "eventData": {
    "amount": 4990,
    "transactionId": "tx_abc123",
    "paidAt": "2024-02-15T10:00:00.000Z"
  },
  "createdAt": "2024-02-15T10:00:00.000Z"
}
```
