# Error Handling Reference

Complete error catalog for the Blowfish Agent API.

## Error Response Format

All errors return:

```json
{
  "error": "Human-readable error message"
}
```

Validation errors also include a `details` array:

```json
{
  "error": "Validation failed",
  "details": [
    { "field": "ticker", "message": "Must be 2-10 uppercase alphanumeric characters" }
  ]
}
```

## HTTP Status Codes

| Status | Meaning | Common causes |
|--------|---------|---------------|
| 400 | Bad Request | Missing required fields, invalid format, validation failure |
| 401 | Unauthorized | Missing auth header, invalid JWT, expired JWT, invalid nonce/signature |
| 403 | Forbidden | Wallet address mismatch |
| 404 | Not Found | Launch event or token doesn't exist |
| 409 | Conflict | Duplicate ticker |
| 500 | Internal Server Error | Unexpected server failure |
| 503 | Service Unavailable | Database or downstream service down |

## Authentication Errors

| Error Message | Status | Cause | Fix |
|---------------|--------|-------|-----|
| `wallet is required` | 400 | Missing wallet in challenge/verify request | Include `wallet` field in request body |
| `Invalid wallet address format` | 400 | Malformed base58 address | Verify the address is a valid Solana public key |
| `signature is required` | 400 | Missing signature in verify request | Include `signature` field |
| `nonce is required` | 400 | Missing nonce in verify request | Include `nonce` field |
| `Invalid or expired nonce` | 401 | Nonce expired (>5 min) or already consumed | Request a new challenge |
| `Invalid signature` | 401 | ed25519 verification failed | Check signing key matches wallet, ensure you're signing the raw nonce bytes |
| `Missing or invalid authorization header` | 401 | No Bearer token on authenticated endpoint | Include `Authorization: Bearer <jwt>` header |
| `Invalid token: missing sub (wallet)` | 401 | JWT payload missing `sub` claim | Re-authenticate to get a valid JWT |
| `Session expired or invalid` | 401 | Per-session secret expired in cache | Re-authenticate from Step 1 |
| `Invalid or expired token` | 401 | JWT verification or expiry failure | Re-authenticate to get a new JWT |

## Launch Errors

| Error Message | Status | Cause | Fix |
|---------------|--------|-------|-----|
| `Validation failed` | 400 | Request body fails Zod schema | Check `details` array for per-field errors |
| `Ticker MCT already exists` | 409 | Ticker is taken | Choose a different ticker |
| `Failed to register launch` | 500 | Queue or database failure | Retry after a brief delay |

## Status Errors

| Error Message | Status | Cause | Fix |
|---------------|--------|-------|-----|
| `Event ID is required` | 400 | Missing eventId path parameter | Include the eventId in the URL |
| `Launch not found` | 404 | No launch with this eventId | Verify the eventId from the launch response |

## Token Query Errors

| Error Message | Status | Cause | Fix |
|---------------|--------|-------|-----|
| `Agent ID is required` | 400 | Agent ID missing from JWT | Re-authenticate |
| `Mint address and agent ID are required` | 400 | Missing mintAddress path parameter | Include mint address in URL |
| `Token not found` | 404 | Token doesn't exist or doesn't belong to agent | Check mint address, ensure it was launched by your agent |

## Fee Claiming Errors

| Error Message | Status | Cause | Fix |
|---------------|--------|-------|-----|
| `Mint address is required` | 400 | Missing mintAddress path parameter | Include mint address in URL |
| `Agent ID is required` | 400 | Agent ID missing from JWT payload | Re-authenticate |
| `Wallet address is required` | 400 | Wallet address missing from JWT payload | Re-authenticate |
| `Failed to create claim transactions` | 400 | On-chain transaction creation failed | Check that the token has claimable fees |
| `Failed to submit signed transaction` | 400 | Signed transaction submission failed | Verify signature is correct; transaction may have expired |
| `Wallet address does not match agent wallet` | 403 | JWT wallet doesn't match the agent's registered wallet | Authenticate with the wallet that launched the token |
| `Token not found for this agent` | 404 | No token with this mint for the agent | Check mint address, ensure your agent launched it |
| `Agent not found` | 404 | Agent record not found in database | Re-authenticate |

## Launch Status Values

These are not HTTP errors but terminal statuses returned by the status polling endpoint:

| Status | Meaning | Resolution |
|--------|---------|------------|
| `failed` | On-chain deployment failed | Retry with a new launch request |
| `rate_limited` | Daily launch limit exceeded | Wait until UTC midnight |

## Retry Strategy

| Error Type | Retryable | Strategy |
|-----------|-----------|----------|
| 400 | No | Fix the request and retry |
| 401 | Yes | Re-authenticate, then retry |
| 403 | No | Use the correct wallet |
| 404 | No | Verify resource identifiers |
| 409 | No | Choose different parameters |
| 5xx | Yes | Exponential backoff, then retry |

## Best Practices

1. **Always validate locally first** — check ticker format (`^[A-Z0-9]+$`) and field lengths before making API calls
2. **Handle 401 gracefully** — re-authenticate automatically when JWT expires
3. **Log error responses** — include the full error message for debugging
4. **Respect the daily limit** — check if you've already launched today before submitting
