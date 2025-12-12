# Shwary Merchant Endpoint Integration

This note is intentionally scoped to the Shwary merchant API.

## How to create marchant Key

1. Create an account at app.shwary.com
2. Navigate to the settings
3. Create merchant key
4. Copy the merchant Key and Merchant ID

NOTE: The merchant key is only visible once, once closed cannot be revealed anymore, you need to make sure to copy it somewhere.

## Merchant Overview

- **Provider**: Shwary (`https://api.shwary.com/`)
- **Purpose**: Initiate mobile money payments for wallets in **DRC**, **Kenya**, and **Uganda**.
- **Merchant endpoint**: `POST https://api.shwary.com/api/v1/merchants/payment/{countryCode}`
- **Minimum amount**: strictly greater than `2 900` (the backend rounds to whole integers).

## Authentication & Headers

Every merchant endpoint is protected by `MerchantGuard`. Requests must include:

| Header           | Description                                    |
| ---------------- | ---------------------------------------------- |
| `x-merchant-id`  | The merchant's UUID                            |
| `x-merchant-key` | Shared secret generated via `POST /merchants`. |

Invalid/missing keys result in `401 Unauthorized`.

## Supported Countries

Use the `countryCode` path parameter to target the right rails.

| Country Code | Country                      | Phone Code | Currency |
| ------------ | ---------------------------- | ---------- | -------- |
| `DRC`        | Democratic Republic of Congo | `+243`     | `CDF`    |
| `KE`         | Kenya                        | `+254`     | `KES`    |
| `UG`         | Uganda                       | `+256`     | `UGX`    |

`clientPhoneNumber` must start with the country’s phone code.

## Shared Request Body

```jsonc
{
  "amount": 5000,
  "clientPhoneNumber": "+243812345678",
  "callbackUrl": "https://merchant.example.com/hooks/shwary" // optional
}
```

| Field               | Required | Description                                                             |
| ------------------- | -------- | ----------------------------------------------------------------------- |
| `amount`            | ✔        | Amount in the target country’s currency. Must be > 0 (>= 2900 for DRC). |
| `clientPhoneNumber` | ✔        | MSISDN in E.164 format with country code.                               |
| `callbackUrl`       | ✖        | HTTPS endpoint that receives async transaction updates.                 |

## Response Shape

All endpoints return a `TransactionResponseDto`:

```json
{
  "id": "c0fdfe50-24be-4de1-9f66-84608fd45a5f",
  "userId": "merchant-uuid",
  "amount": 5000,
  "currency": "CDF",
  "type": "deposit",
  "status": "pending",
  "recipientPhoneNumber": "+243812345678",
  "referenceId": "merchant-6c661f48-0c39-4474-9621-931d4419babb",
  "metadata": null,
  "failureReason": null,
  "completedAt": null,
  "createdAt": "2025-01-16T10:15:00.000Z",
  "updatedAt": "2025-01-16T10:15:00.000Z",
  "isSandbox": false
}
```

- `status` starts as `pending` for real-money flows and `completed` for sandbox.
- `isSandbox` lets you distinguish simulated transactions across the stack.

## Callback Contract

If `callbackUrl` is provided, Shwary performs a POST with the transaction payload whenever the status changes (e.g., from `pending` → `completed` or `failed`). Expect the same JSON as the response body plus contextual fields such as `error` when failures happen.

Retries are **not** automatic today; ensure your endpoint is highly available and idempotent.

## Endpoints

### 1. Standard Merchant Payment

`POST /merchants/payment/:countryCode`

Performs a payment using the customer’s mobile-money account and settles into the merchant’s Shwary wallet.

- **Status Flow**: `pending` → `completed` (after Pretium callback) or `failed`.

**Sample Request**

```http
POST /merchants/payment/DRC HTTP/1.1
Host: api.shwary.com
x-merchant-id: f5a9f5db-1b33-4d76-9168-0035a6f71170
x-merchant-key: shwary_live_merchant_secret
Content-Type: application/json

{
  "amount": 5000,
  "clientPhoneNumber": "+243812345678",
  "callbackUrl": "https://merchant.example.com/hooks/shwary"
}
```

**Typical Response**

```json
{
  "id": "e44b497a-d2d3-4b82-aad5-0bddb674ba69",
  "status": "pending",
  "currency": "CDF",
  "pretium_transaction_id": "PRT-123456",
  "isSandbox": false,
  "...": "..."
}
```

### 2. Sandbox Merchant Payment

`POST /merchants/payment/sandbox/:countryCode`

Designed for integration tests and UAT. No blockchain or mobile-money calls occur; a completed transaction row is created with `is_sandbox = true`.

- **Status Flow**: `completed` instantly (or `failed` if validation fails).
- **Callback**: Still fires so you can test webhook handling.

**Sample Response**

```json
{
  "id": "6a6bb0e6-7400-4bd9-9b1e-5b518c352da9",
  "status": "completed",
  "currency": "CDF",
  "referenceId": "merchant-sandbox-5f6b8f7a-4430-4e2a-ab46-0b3bf63503d4",
  "isSandbox": true,
  "...": "..."
}
```

Use sandbox to validate:

1. Header signing and authentication.
2. Payload validation errors (amount rules, phone formats, etc.).
3. Callback endpoint behavior.

## Error Handling

| Status             | Meaning                                                   | Example                                                 |
| ------------------ | --------------------------------------------------------- | ------------------------------------------------------- |
| `400 Bad Request`  | Validation failures (amount, phone, unsupported country). | `{ "message": "Amount must be greater than 2900 CDF" }` |
| `401 Unauthorized` | Missing/invalid merchant headers.                         | `{ "message": "Invalid merchant key" }`                 |
| `404 Not Found`    | Client or merchant not registered in Shwary.              | `{ "message": "Client not found" }`                     |
| `502 Bad Gateway`  | Gateway failure propagating back to the merchant.         |

Always inspect `failureReason` (and `error` field in callbacks) to diagnose issues.

## Integration Checklist

1. Call the sandbox endpoint to validate auth, payloads, and callback handling.
2. Listen for webhook updates and reconcile using `transactionId`.
3. Launch with the standard or Shwary endpoint once you’re confident.

Need help? Share request IDs timestamps when contacting Shwary support. This speeds up log correlation on our side.

## Example request (cURL)

The snippet below triggers a CDF payment for a DRC wallet. Swap `DRC` with `KE` or `UG`, and adjust amount/currency to match the target market.

```bash
curl -X POST "https://api.shwary.com/api/v1/merchants/payment/DRC" \
  -H "Content-Type: application/json" \
  -H "x-merchant-key: $SHWARY_MERCHANT_KEY" \
  -H "x-merchant-id: $SHWARY_MERCHANT_ID" \
  -d '{
    "amount": 5000,
    "clientPhoneNumber": "+243820000000",
    "callbackUrl": "https://your-app.com/api/shwary/callback"
  }'
```
