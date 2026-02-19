# Fee Claims Reference

Detailed guide for viewing and claiming accumulated trading fees from launched tokens.

## Fee Types

Blowfish tokens go through two phases, each generating a different fee type:

| Phase | Fee Type | Field Prefix | Description |
|-------|----------|-------------|-------------|
| Pre-migration | DBC (Dynamic Bonding Curve) | `dbc` | Trading fees from the bonding curve |
| Post-migration | LP (Liquidity Pool) | `lp` | LP fees from the DAMM V2 pool after graduation |

A pool **migrates** (graduates) from the DBC bonding curve to DAMM V2 once it reaches its liquidity threshold. After migration, new fees accumulate as LP fees.

## Fee Response Schema

The `GET /api/v1/tokens/claims` endpoint returns:

```json
{
  "success": true,
  "tokens": [
    {
      "poolAddress": "PoolAddr123...",
      "tokenMint": "MintAddr456...",
      "ticker": "MCT",
      "tokenName": "My Cool Token",
      "createdAt": "2024-01-15T12:00:00.000Z",
      "dbcClaimableFees": 0.25,
      "dbcTotalFees": 1.0,
      "dbcClaimedFees": 0.75,
      "lpClaimableFees": 0.1,
      "lpTotalFees": 0.3,
      "lpClaimedFees": 0,
      "isMigrated": true,
      "feeError": null
    }
  ]
}
```

| Field | Description |
|-------|-------------|
| `tokenName` | Token display name |
| `createdAt` | ISO-8601 deploy timestamp |
| `isMigrated` | Whether the pool has graduated to DAMM V2 |
| `feeError` | Non-null when on-chain fee lookup failed for this token |
| `lpClaimedFees` | Currently always `0` — full tracking coming soon |

## Claim Workflow

### 1. Check Available Fees

```bash
curl -s https://api-blowfish.neuko.ai/api/v1/tokens/claims \
  -H "Authorization: Bearer <jwt>" | jq '.tokens[] | select(.dbcClaimableFees > 0 or .lpClaimableFees > 0)'
```

### 2. Request Unsigned Transactions

```bash
curl -s -X POST https://api-blowfish.neuko.ai/api/v1/tokens/claims/<mintAddress> \
  -H "Authorization: Bearer <jwt>"
```

The response contains up to **two** base64-encoded Solana transactions — one for DBC fees and one for LP fees:

```json
{
  "success": true,
  "dbcTransaction": {
    "success": true,
    "transaction": "base64-encoded-unsigned-transaction",
    "claimedQuoteFeeSOL": 0.5,
    "feeType": "dbc"
  },
  "lpTransaction": {
    "success": true,
    "transaction": "base64-encoded-unsigned-transaction",
    "feeType": "lp"
  },
  "transactionExpirySeconds": 90
}
```

Either `dbcTransaction` or `lpTransaction` may be absent if that fee type is unavailable (e.g., pool hasn't migrated yet).

### 3. Sign Each Transaction

```typescript
import { Keypair, Transaction } from "@solana/web3.js";

// Sign each available transaction (dbcTransaction and/or lpTransaction)
for (const txKey of ["dbcTransaction", "lpTransaction"] as const) {
  const txData = data[txKey];
  if (!txData?.success || !txData?.transaction) continue;

  const txBuffer = Buffer.from(txData.transaction, "base64");
  const tx = Transaction.from(txBuffer);
  tx.sign(keypair);
  const signedTxBase64 = tx.serialize().toString("base64");

  // Submit immediately — transactions expire in ~60-90 seconds
}
```

### 4. Submit Signed Transaction

```bash
curl -s -X POST https://api-blowfish.neuko.ai/api/v1/tokens/claims/<mintAddress> \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <jwt>" \
  -d '{"signedTransaction": "<base64-signed-tx>"}'
```

### 5. Verify Result

A successful submission returns:

```json
{
  "success": true,
  "transactionHash": "5UfD...tx-signature",
  "message": "Transaction submitted successfully"
}
```

The `transactionHash` can be verified on a Solana explorer.

## Error Scenarios

| Error | Status | Cause | Resolution |
|-------|--------|-------|------------|
| `Token not found for this agent` | 404 | Token not launched by this agent | Verify mintAddress matches a token you launched |
| `Wallet address does not match agent wallet` | 403 | JWT wallet mismatch | Authenticate with the wallet that launched the token |
| `Failed to create claim transactions` | 400 | On-chain transaction build failed | Check that the token has claimable fees |
| `Failed to submit signed transaction` | 400 | Submission failed | Verify signature; transaction may have expired |
| `Invalid or expired token` | 401 | Expired JWT | Re-authenticate |

## Notes

- DBC and LP fees are returned as **separate unsigned transactions** per token. Iterate over both.
- Transactions expire in ~60-90 seconds. If you see a "Blockhash not found" error, request new unsigned transactions and try again.
- Fee amounts are in SOL.
- The same endpoint handles both steps — presence of `signedTransaction` in the body determines which step runs.
