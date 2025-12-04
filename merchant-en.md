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
- **Merchant endpoint**: `POST https://api.shwary.com/api/v1/merchants/payment/shwary/{countryCode}`
- **Headers required**:
  - `x-merchant-key`
  - `x-merchant-id`
- **Minimum amount**: strictly greater than `2 900` (the backend rounds to whole integers).

## Environment Variables

```env
SHWARY_MERCHANT_KEY=your-shwary-merchant-key
SHWARY_MERCHANT_ID=your-shwary-merchant-id
```

These values are read by `app/api/shwary/route.ts` when we proxy calls to Shwary.

## Callback Endpoint (`POST /api/shwary/callback`)

Shwary sends asynchronous updates to the callback URL specified above. The backend:

1. Accepts the JSON payload (no authentication headers are required by Shwary).
2. Locates the matching order using either `metadata.orderId` or `transactionId`.
3. Maps Shwary’s status (`PENDING`, `COMPLETED`, `FAILED`) to the internal order status (`pending`, `completed`, `cancelled`).
4. Updates the Convex order record via `api.orders.updateOrderStatus`.

Expected callback states:

- **Pending**: initial push request (user still needs to confirm on their phone).
- **Completed**: customer entered their PIN and funds cleared.
- **Failed**: customer rejected the prompt, the push timed out, or Shwary rejected it.

We always respond with `200` + CORS headers so Shwary treats the callback as delivered.

## Validation Summary

1. Amount must be numeric and `2900 CDF` for DRC, `3500 UGX` for Uganda and `130 KES` for Kenya.
2. Phone number must be provided.
3. `countryCode` limited to `DRC`, `KE`, `UG`.
4. Metadata (if present) must be a JSON object.
5. Merchant headers (`SHWARY_MERCHANT_KEY`, `SHWARY_MERCHANT_ID`) are mandatory; missing values result in `401`.
