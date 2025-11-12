# Revio Pay API - Relevant Features for Backend Design

## Overview
Revio Pay is a payment orchestration platform that provides a unified API for payment processing, supporting multiple payment methods and gateways.

**Base URL**: `https://gate.reviopay.com/api/v1/`

---

## Authentication

### API Keys
- **Method**: Bearer token authentication
- **Header**: `Authorization: Bearer <secret_key>`
- **Location**: API keys obtained from Developers section in Revio dashboard
- **Requirements**: Brand ID + API secret key for all requests

---

## Core API Endpoints

### 1. Payment Processing (Deposits)

#### Create Purchase
- **Endpoint**: `POST /purchases/`
- **Purpose**: Initiate a payment/deposit
- **Returns**: `checkout_url` for hosted payment page
- **Supports**: Multiple payment methods (card, instant EFT, debit order, QR, etc.)

#### Retrieve Purchase Status
- **Endpoint**: `GET /purchases/{id}/`
- **Purpose**: Check payment status
- **Use Case**: Verify if deposit completed successfully

#### Charge with Token
- **Endpoint**: `POST /purchases/{id}/charge/`
- **Purpose**: Charge using saved payment token
- **Use Case**: Recurring deposits without re-entering payment details

#### Capture Payment
- **Endpoint**: `POST /purchases/{id}/capture/`
- **Purpose**: Capture previously authorized payment
- **Use Case**: Two-phase payment processing

#### Cancel Purchase
- **Endpoint**: `POST /purchases/{id}/cancel/`
- **Purpose**: Cancel pending purchase
- **Use Case**: User cancellation before payment completes

### 2. Payouts (Withdrawals)

#### Create Payout
- **Endpoint**: `POST /payouts/`
- **Purpose**: Initiate withdrawal/payout to user
- **Status**: Creates unprocessed payout

#### Retrieve Payout
- **Endpoint**: `GET /payouts/{id}/`
- **Purpose**: Check payout status
- **Use Case**: Track withdrawal processing

#### List Payout Methods
- **Endpoint**: `GET /payout_methods/`
- **Purpose**: Get available withdrawal methods
- **Returns**: Supported payout channels

### 3. Refunds

#### Refund Purchase
- **Endpoint**: `POST /purchases/{id}/refund/`
- **Purpose**: Refund a paid purchase
- **Use Case**: Process refunds for failed transactions or disputes

---

## Webhooks

### Webhook Management

#### Create Webhook
- **Endpoint**: `POST /webhooks/`
- **Purpose**: Register webhook endpoint

#### List Webhooks
- **Endpoint**: `GET /webhooks/`
- **Purpose**: View configured webhooks

#### Webhook Deliveries
- **Endpoint**: `GET /webhooks/deliveries/`
- **Purpose**: Audit trail of webhook delivery attempts
- **Use Case**: Debugging webhook issues

### Webhook Events

#### Payment Events
- `purchase.created` - Invoice/purchase created
- `purchase.paid` - Payment successful
- `purchase.payment_failure` - Payment failed

#### Additional Events
- Subscription charges
- Pending payment status transitions
- Supports all payment method types

### Webhook Security

#### Signature Verification
- **Algorithm**: RSA PKCS#1 v1.5 with SHA256
- **Header**: `X-Signature` (base64-encoded signature)
- **Verification**: Compare signature against hash of request body
- **Public Key**: Available via `Webhook.public_key` or `/public_key/` endpoint

#### Retry Logic
- **Strategy**: Exponential backoff
- **Final Attempt**: +36 hours from first attempt
- **Use Case**: Handle temporary endpoint failures

---

## Client Management

### Client Endpoints

#### Create Client
- **Endpoint**: `POST /clients/`
- **Purpose**: Register user/customer

#### List Clients
- **Endpoint**: `GET /clients/`
- **Purpose**: Retrieve customer list

#### Get Client Details
- **Endpoint**: `GET /clients/{id}/`
- **Purpose**: Get specific customer data

#### Client Transaction Feed
- **Endpoint**: `GET /clients/{id}/feed/`
- **Purpose**: Transaction history for specific client
- **Use Case**: Display user's deposit/withdrawal history

#### Recurring Tokens
- **Endpoint**: `GET /clients/{id}/recurring_tokens/`
- **Purpose**: List saved payment methods for client
- **Use Case**: Allow users to manage saved payment methods

---

## Balance & Account Management

### Balance Endpoints

#### Get Balance
- **Endpoint**: `GET /balance/`
- **Purpose**: Retrieve balance data with bank accounts
- **Use Case**: Check merchant/platform balance

#### Company Balance
- **Endpoint**: `GET /account/json/balance/`
- **Purpose**: Get company balance summary

#### Company Turnover
- **Endpoint**: `GET /account/json/turnover/`
- **Purpose**: Retrieve turnover metrics

### Statements

#### Generate Statement
- **Endpoint**: `POST /company_statements/`
- **Purpose**: Create financial statement

#### List Statements
- **Endpoint**: `GET /company_statements/`
- **Purpose**: Retrieve historical statements

---

## Payment Methods

### Available Methods
- **Endpoint**: `GET /payment_methods/`
- **Returns**: List of supported payment methods
- **Types**: Card, Instant EFT, Capitec Pay, Debit Order, QR codes, etc.

---

## Billing & Invoicing

### One-Time Billing
- **Endpoint**: `POST /billing/`
- **Purpose**: Send one-time invoices

### Billing Templates
- **Endpoints**:
  - `POST /billing_templates/` - Create template
  - `GET /billing_templates/` - List templates
  - `GET /billing_templates/{id}/` - Get template
  - `PUT /billing_templates/{id}/` - Update template
  - `DELETE /billing_templates/{id}/` - Delete template
  - `POST /billing_templates/{id}/send_invoice/` - Send invoice
  - `POST /billing_templates/{id}/add_subscriber/` - Add subscriber

---

## Payment Flows

### 1. Redirect Flow (Standard)
1. Backend creates purchase via `POST /purchases/`
2. User redirected to `checkout_url`
3. User completes payment on Revio-hosted page
4. User redirected back via `success_redirect` or `failure_redirect`
5. Backend receives webhook notification
6. Backend queries `GET /purchases/{id}/` to verify status

### 2. Server-to-Server (API Only)
1. Backend creates purchase via `POST /purchases/`
2. Backend polls `GET /purchases/{id}/` for status
3. Backend processes webhook for async confirmation

### 3. Tokenized/Recurring
1. User saves payment method (generates token)
2. Backend charges using `POST /purchases/{id}/charge/`
3. No user interaction required for subsequent payments

---

## Key Features for Your Design

### For Deposits
- ✅ Create purchase endpoint
- ✅ Webhook notifications for payment success/failure
- ✅ Status verification endpoint
- ✅ Tokenization for saved payment methods
- ✅ Multiple payment method support

### For Withdrawals
- ✅ Payout creation endpoint
- ✅ Payout status tracking
- ✅ Multiple payout method support

### For Transaction Management
- ✅ Client transaction feed (history)
- ✅ Refund capabilities
- ✅ Webhook delivery audit trail

### For Security
- ✅ RSA signature verification for webhooks
- ✅ API key authentication
- ✅ Public key endpoint for verification
- ✅ Webhook delivery logging

### For User Management
- ✅ Client CRUD operations
- ✅ Saved payment method management
- ✅ Transaction history per client

---

## Important Considerations

### Webhook Handling
- Must verify `X-Signature` header using RSA-SHA256
- Store webhook delivery logs for audit
- Implement idempotency (webhooks may retry)
- Handle all three states: created, paid, payment_failure

### Concurrency & Idempotency
- Same purchase ID should not be processed twice
- Use database transactions to prevent race conditions
- Store Revio purchase/payout IDs to track status

### Error Handling
- API may return errors during purchase creation
- Webhooks may be delayed or arrive out of order
- Need to handle Revio downtime scenarios
- Implement retry logic for failed API calls

### Payment States
- **created** → Initiated but not paid
- **paid** → Successfully completed
- **payment_failure** → Failed transaction
- **pending** → Awaiting processing (some payment methods)
- **cancelled** → User or system cancelled

---

## Documentation Resources

- Main Docs: https://docs.reviopay.com/
- API Swagger: https://gate.reviopay.com/api/
- Webhook Guide: https://docs.reviopay.com/docs/using-webhooks
- Payment Flow: https://docs.reviopay.com/docs/standard-payment-flow
- API Keys: https://docs.reviopay.com/docs/api-keys
