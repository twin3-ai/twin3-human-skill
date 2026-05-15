# Twin3 Human API Reference

Detailed parameter, request, response, and error reference for `GET https://human.twin3.ai/v1/human`.

This file is loaded by the AI agent on demand when more detail is needed beyond the high-level command list in `SKILL.md`.

---

## Endpoint

```
GET https://human.twin3.ai/v1/human
```

| Aspect | Value |
|---|---|
| Method | `GET` (always) |
| Auth | None — payment-gated via x402, not an API key |
| Content-Type | None on request (query-string only); `application/json` on response |
| Idempotency | Safe — same query always returns the current humanity verdict; no side effects beyond the payment settle |

## Payment

Every call requires an x402 micropayment. The server returns HTTP 402 on the first request with a `payment-required` header listing the accepted payment methods, and HTTP 200 on the retry with a valid `PAYMENT-SIGNATURE` header.

### Supported payment methods

| Network | Network ID (CAIP-2) | Token | Token Contract | Amount per call |
|---|---|---|---|---|
| **X Layer** (recommended, zero gas) | `eip155:196` | USD₮0 | `0x779ded0c9e1022225f8e0630b35a9b54be713736` | 1000 (= $0.001) |
| Base | `eip155:8453` | USDC | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` | 1000 (= $0.001) |

For OKX Onchain OS agents, **always select X Layer** — the OKX broker sponsors gas, so the buyer's total cost is exactly $0.001 USD₮0 with no native-coin requirement.

For non-OKX agents (or fallback): Base USDC works as well, via the Coinbase CDP x402 facilitator path.

### Schemes

- `exact` — single-shot payment, EIP-3009 `transferWithAuthorization` under the hood. Used by both Base and X Layer accept entries.
- `aggr_deferred` — OKX extension for batched / session payments. Not enabled on this endpoint as of v0.1.0; will be added when Twin3 supports session-priced bulk queries (~Q3 2026).

## Query Parameters

### `wallet` *(required)*

The EVM address to verify.

- **Format**: `0x` prefix followed by 40 hex characters (case-insensitive per EIP-55, lowercase or mixed-case both accepted).
- **Pattern**: `^0x[a-fA-F0-9]{40}$`
- **Returns HTTP 400 `invalid_wallet`** if the format is invalid.

### `min_score` *(optional)*

Integer threshold 0-255. When present, the response includes `passes: bool` indicating whether `score >= min_score`.

- **Format**: integer string `0` through `255`.
- **Pattern**: `^([0-9]|[1-9][0-9]|1[0-9][0-9]|2[0-4][0-9]|25[0-5])$`
- **Returns HTTP 400 `invalid_min_score`** if out of range.

**Reference levels** (the canonical SCORING_WEIGHTS sum to 255):

| Weight | Verification |
|---|---|
| 15 | reCAPTCHA |
| 30 | Google OAuth |
| 25 | Discord |
| 10 | LINE |
| 30 | Telegram |
| 55 | Apple ID |
| 90 | 2FA (g2fa) |
| **255** | All of the above combined |

### `check` *(optional)*

Comma-separated list of verification method names. When present, the response includes `verifications: {method: bool}` answering yes/no for ONLY the methods the buyer specifically named (ZK-style — Twin3 never enumerates the full set).

- **Valid method names**: `recaptcha`, `google`, `discord`, `line`, `telegram`, `apple`, `g2fa`, `onchain`, `worldid`.
- **Returns HTTP 400 `invalid_check`** if any name is not in the valid list (case-insensitive).

### `include` *(optional)*

Comma-separated CSV of richer fields to add to the response:

| Token | Adds to response | Notes |
|---|---|---|
| `verified_at` | `verifiedAt` (ISO-8601 string) | SBT mint timestamp; useful for sybil-pattern detection (mass mints in a short window) |
| `dimensions_filled` | `dimensionsFilled` (int 0-256) | Count of non-zero bytes in the wallet's Twin Matrix; signals profile completeness |

Unknown tokens return HTTP 400 `invalid_include`.

## Response (HTTP 200)

```json
{
  "wallet": "0x344659f3ef3c2d2a0cdf071ea13fa87867777777",
  "isHuman": true,
  "score": 105,
  "scoreFloor": 15,
  "scoreSource": "firestore",
  "sbtId": 142031,
  "queriedAt": "2026-05-11T14:51:06.979Z",

  "passes": true,                  // present only if min_score was supplied
  "verifications": {               // present only if check= was supplied
    "telegram": true,
    "apple": false
  },
  "verifiedAt": "2024-09-12T03:14:22Z",  // present only if include=verified_at
  "dimensionsFilled": 8            // present only if include=dimensions_filled
}
```

| Field | Type | Notes |
|---|---|---|
| `wallet` | string (0x…40hex, lowercase) | Echo of the input wallet, normalized |
| `isHuman` | boolean | True if a Twin3 humanity verification exists for this wallet (i.e. SBT minted) |
| `score` | integer 0-255 | Humanity score (sum of verification weights) |
| `scoreFloor` | integer | Baseline weight (15 = reCAPTCHA) — useful for buyers to interpret raw scores |
| `scoreSource` | string | `"firestore"` (canonical registry) or `"chain"` (fallback to on-chain SBT byte 0) |
| `sbtId` | integer or null | Twin Matrix SBT id on BNB Chain; null if wallet has no SBT |
| `queriedAt` | string (ISO-8601 UTC) | When this verdict was computed (now) |

## Response (HTTP 4xx errors)

```json
{ "error": "invalid_wallet" }                      // 400 — wallet param malformed
{ "error": "invalid_min_score", "message": "..." } // 400 — min_score not 0-255
{ "error": "invalid_check", "message": "..." }     // 400 — unknown method name
{ "error": "invalid_include", "message": "..." }   // 400 — unknown include token
{ "error": "internal_error" }                      // 500 — backend issue, retry safe
```

The 402 response (before payment) carries the standard x402 challenge:
v2 puts it in the `PAYMENT-REQUIRED` response header (base64-encoded
JSON); v1 puts an `x402Version` body. The buyer does not parse or pay
this manually — the **OKX Agent Payments Protocol**
(`okx-agent-payments-protocol` skill) detects the 402, decodes the
`accepts` array, signs the EIP-3009 authorization, attaches the
`PAYMENT-SIGNATURE` (v2) / `X-PAYMENT` (v1) header, and replays the
request automatically.

## Score Source — `firestore` vs `chain`

Twin3 maintains two layers of humanity data:

1. **`firestore`** — the canonical off-chain registry. Updated in real time as users complete new verifications. This is the source we use whenever available; it includes off-chain verifications (Apple ID, 2FA) that the chain mirror may lag on.
2. **`chain`** — fallback when Firestore is unavailable. Reads byte 0 of the Twin Matrix SBT's `tokenURI()` SVG, which is the humanity-score byte. Slower to update (the on-chain byte only mirrors after periodic batch jobs).

The `scoreSource` field tells the buyer which path resolved the score. For most wallets, `firestore` is authoritative.

## ZK-Style Verification: Why and How

The `check=` parameter is intentionally **buyer-named, not enumerated**. The server returns the verdict only for the specific methods the buyer asks about. This protects user privacy in two ways:

1. **The full verification list never leaves Twin3**. If a wallet has 7 verifications (telegram, apple, g2fa, google, discord, line, recaptcha), querying only `check=telegram,apple` returns `{telegram: true, apple: true}` — the buyer never learns the other 5 were also cleared.
2. **The buyer must already know what to ask about**. A drive-by snooping query can't iterate the full method list to fingerprint a user; they would need 9 separate paid queries to confirm absence of every method.

For wallets WITHOUT a Twin3 SBT, all `check=` queries return false for every named method, and `isHuman` is false.

## Sybil-Filter Pseudocode

For agents that need to filter a large wallet list:

```
INPUT: array of wallets, threshold (e.g. 70)
OUTPUT: array of "qualified human" wallets

CONFIRM total cost = wallets.length × $0.001
  if > $1 (i.e. > 1000 wallets), ask user to approve first

FOR each wallet in wallets:
  result = call /v1/human?wallet=<wallet>&min_score=<threshold>
  if result.passes == true:
    qualified.push(wallet)

RETURN qualified
```

The agent should batch these in parallel when latency matters (X Layer broker handles concurrent settles fine; rate-limit yourself to 10 concurrent calls to be safe).

## Rate Limits

- Per-IP rate limit: 600 requests / minute (express-rate-limit default).
- Per-buyer-wallet rate limit: not enforced today; OKX broker may apply its own.
- For batch sybil filtering, stay below 10 parallel queries to avoid 429 from upstream rate limiter.

## Versioning

This is `twin3-human` v0.1.0. The `/v1/human` endpoint URL is stable; future versions will use `/v2/...` paths and a new skill version.

## See Also

- [Quickstart for OKX agents](./quickstart-okx.md) — concrete end-to-end run (plain GET; 402 auto-paid by the OKX Agent Payments Protocol).
- Twin3 product brief: [https://twin3.ai](https://twin3.ai)
- Twin Matrix SBT on BNB Chain: contract `0xE3ec133e29adDfbBA26a412c38ed5De37195156f`
