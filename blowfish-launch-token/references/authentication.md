# Authentication Reference

Complete wallet-based challenge-response authentication for the Blowfish Agent API.

## Flow Summary

```
Agent                              API
  |                                 |
  |-- POST /api/auth/challenge ---->|  (send wallet address)
  |<-------- { nonce } -------------|  (5-min expiry)
  |                                 |
  |  sign(nonce)                     |
  |                                 |
  |-- POST /api/auth/verify ------->|  (wallet + nonce + sig)
  |<-------- { token } -------------|  (15-min JWT)
  |                                 |
  |-- Any API call --------------->|
  |   Authorization: Bearer <jwt>   |
```

## TypeScript Implementation

```typescript
import { Keypair } from "@solana/web3.js";
import nacl from "tweetnacl";
import bs58 from "bs58";

const BASE_URL = "https://api-blowfish.neuko.ai";

async function authenticate(keypair: Keypair): Promise<string> {
  const wallet = keypair.publicKey.toBase58();

  // Step 1: Request challenge nonce
  const challengeRes = await fetch(`${BASE_URL}/api/auth/challenge`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ wallet }),
  });
  const { nonce } = await challengeRes.json();

  // Step 2: Sign the raw nonce
  const nonceBytes = new TextEncoder().encode(nonce);
  const signature = nacl.sign.detached(nonceBytes, keypair.secretKey);
  const signatureBase58 = bs58.encode(signature);

  // Step 3: Verify and receive JWT
  const verifyRes = await fetch(`${BASE_URL}/api/auth/verify`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ wallet, nonce, signature: signatureBase58 }),
  });
  const { token } = await verifyRes.json();

  return token; // Valid for 15 minutes
}
```

## Dependencies

- `@solana/web3.js` — Solana keypair and public key utilities
- `tweetnacl` — ed25519 detached signatures
- `bs58` — Base58 encoding for signatures

## Bash Implementation

For agents that prefer shell scripts:

```bash
#!/bin/bash
# Requires: solana CLI, jq, base58 encoder

WALLET=$(solana address)
BASE="https://api-blowfish.neuko.ai"

# Step 1: Get challenge
NONCE=$(curl -s -X POST "$BASE/api/auth/challenge" \
  -H "Content-Type: application/json" \
  -d "{\"wallet\": \"$WALLET\"}" | jq -r '.nonce')

# Step 2: Sign the raw nonce
# Sign $NONCE using solana CLI or equivalent ed25519 tool
# The signature must be base58-encoded

# Step 3: Verify
TOKEN=$(curl -s -X POST "$BASE/api/auth/verify" \
  -H "Content-Type: application/json" \
  -d "{\"wallet\": \"$WALLET\", \"nonce\": \"$NONCE\", \"signature\": \"$SIGNATURE\"}" \
  | jq -r '.token')

echo "$TOKEN"
```

## Token Lifecycle

| Property | Value |
|----------|-------|
| Challenge nonce TTL | 5 minutes |
| JWT TTL | 15 minutes |
| Signature algorithm | ed25519 (detached) |
| Signature encoding | base58 |
| Header format | `Authorization: Bearer <jwt>` |

## Re-authentication

When a 401 response is received, the JWT has expired. Re-run the full challenge-response flow to obtain a new token. Do not cache nonces — always request a fresh one.
