---
name: blowfish-launch-token
description: Launch Solana tokens programmatically via the Blowfish Agent API. Use this skill when an agent needs to create new SPL tokens on Solana, check token deployment status, list deployed tokens, view claimable fees, or claim accumulated trading fees.
metadata: {"neuko": {"emoji": "🐡", "homepage": "https://docs.blowfish.neuko.ai", "requires": {"bins": ["curl", "jq"]}}}
---

# Blowfish Token Launch

Launch Solana tokens via the Blowfish Agent API at `https://api-blowfish.neuko.ai`.

## Overview

This skill enables autonomous token launches on Solana through a simple REST API. The flow is:

1. **Authenticate** — wallet-based challenge-response to obtain a JWT
2. **Launch** — submit token parameters (async, returns an eventId)
3. **Poll** — check deployment status until terminal state
4. **Manage** — list tokens, view fees, claim earnings

## Authentication

All endpoints except `/health` require a Bearer JWT obtained via the challenge-response flow.

### Step 1: Request Challenge

```bash
curl -s -X POST https://api-blowfish.neuko.ai/api/auth/challenge \
  -H "Content-Type: application/json" \
  -d '{"wallet": "<WALLET_PUBLIC_KEY>"}'
```

**Response:**
```json
{"nonce": "<random-nonce>"}
```

The nonce expires after **5 minutes**.

### Step 2: Sign the Challenge

Sign the **raw nonce** with the wallet's ed25519 keypair. Encode the signature as **base58**.

### Step 3: Verify and Get JWT

```bash
curl -s -X POST https://api-blowfish.neuko.ai/api/auth/verify \
  -H "Content-Type: application/json" \
  -d '{"wallet": "<WALLET_PUBLIC_KEY>", "nonce": "<nonce>", "signature": "<base58-signature>"}'
```

**Response:**
```json
{"token": "<jwt>"}
```

The JWT is valid for **15 minutes**. Use it in all subsequent requests:

```
Authorization: Bearer <jwt>
```

See [references/authentication.md](references/authentication.md) for the complete TypeScript implementation.

## Launch a Token

```bash
curl -s -X POST https://api-blowfish.neuko.ai/api/v1/tokens/launch \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <jwt>" \
  -d '{
    "name": "My Token",
    "ticker": "MYTKN",
    "description": "A description of the token",
    "imageUrl": "https://example.com/logo.png"
  }'
```

### Parameters

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `name` | string | yes | 1–255 characters |
| `ticker` | string | yes | 2–10 chars, uppercase alphanumeric only |
| `description` | string | no | max 1000 characters |
| `imageUrl` | string | no | valid URL, max 255 characters |

### Response

```json
{
  "success": true,
  "message": "We will notify you when your token is deployed",
  "eventId": "<uuid>"
}
```

### Constraints

- **Rate limit**: 1 launch per agent per UTC day (resets at midnight UTC)
- **Ticker uniqueness**: each ticker can only be used once globally

## Check Deployment Status

```bash
curl -s https://api-blowfish.neuko.ai/api/v1/tokens/launch/status/<eventId> \
  -H "Authorization: Bearer <jwt>"
```

**Response:**
```json
{
  "status": "pending | success | failed | rate_limited",
  "processedAt": "<ISO-8601 or null>"
}
```

Poll every **5–10 seconds** until status is `success`, `failed`, or `rate_limited`.

## List Your Tokens

```bash
curl -s https://api-blowfish.neuko.ai/api/v1/tokens/ \
  -H "Authorization: Bearer <jwt>"
```

**Response:**
```json
{
  "tokens": [
    {
      "poolAddress": "string",
      "tokenMint": "string",
      "ticker": "string",
      "tokenName": "string",
      "isClaimed": false,
      "claimedAt": null,
      "claimedToAddress": null,
      "deployedAt": "2026-01-15T12:00:00Z"
    }
  ]
}
```

## Get a Specific Token

```bash
curl -s https://api-blowfish.neuko.ai/api/v1/tokens/<mintAddress> \
  -H "Authorization: Bearer <jwt>"
```

Returns a single token object matching the structure above.

## View Claimable Fees

```bash
curl -s https://api-blowfish.neuko.ai/api/v1/tokens/claims \
  -H "Authorization: Bearer <jwt>"
```

**Response:**
```json
{
  "claims": [
    {
      "tokenMint": "string",
      "poolAddress": "string",
      "ticker": "string",
      "dbcClaimableFees": 0.5,
      "dbcTotalFees": 1.0,
      "dbcClaimedFees": 0.5,
      "lpClaimableFees": 0.25,
      "lpTotalFees": 0.5,
      "lpClaimedFees": 0.25,
      "isMigrated": false
    }
  ]
}
```

## Claim Fees

Two-step process — get the unsigned transaction, sign it locally, then submit.

### Step 1: Get Unsigned Transaction

```bash
curl -s -X POST https://api-blowfish.neuko.ai/api/v1/tokens/claims/<mintAddress> \
  -H "Authorization: Bearer <jwt>"
```

**Response:**
```json
{
  "success": true,
  "transaction": "<base64-encoded-unsigned-transaction>"
}
```

### Step 2: Submit Signed Transaction

Sign the transaction with the wallet keypair, then submit:

```bash
curl -s -X POST https://api-blowfish.neuko.ai/api/v1/tokens/claims/<mintAddress> \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <jwt>" \
  -d '{"signedTransaction": "<base64-encoded-signed-transaction>"}'
```

**Response:**
```json
{
  "success": true,
  "transactionHash": "string",
  "claimedSOL": 0.75
}
```

See [references/fee-claims.md](references/fee-claims.md) for detailed walkthrough.

## Health Check

```bash
curl -s https://api-blowfish.neuko.ai/health
```

No authentication required.

```json
{"ok": true, "db": "connected", "timestamp": "2026-01-15T12:00:00Z"}
```

## Error Handling

All errors return: `{"error": "Human-readable message"}`

| Status | Scenario | Action |
|--------|----------|--------|
| 400 | Invalid request body | Validate fields against constraints above |
| 401 | Missing or expired JWT | Re-authenticate for a fresh token |
| 404 | Event or token not found | Verify the eventId or mintAddress |
| 409 | Duplicate ticker | Choose a different ticker |
| 429 | Rate limited | Wait until UTC midnight to retry |

See [references/error-handling.md](references/error-handling.md) for full error catalog.
