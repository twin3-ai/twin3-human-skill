# Quickstart — Twin3 Human via OKX Agentic Wallet

End-to-end walkthrough for an OKX Onchain OS agent to verify a wallet's
humanity score using Twin3.

This guide assumes the agent runs inside an Agentic Wallet session
(email-login + TEE key) with `okx/onchainos-skills` installed. If you're
a developer wiring this up from a non-OKX environment, see
[api.md](./api.md) for the raw HTTP shape.

> **Payment model — read this first.** Twin3 Human is a normal
> x402-gated HTTP resource. The agent does **not** call a payment CLI
> for it. The agent issues a plain `GET`; the first response is
> `HTTP 402` with a `PAYMENT-REQUIRED` challenge; the **OKX Agent
> Payments Protocol** (`okx-agent-payments-protocol` skill) auto-detects
> that 402, signs the $0.001 USD₮0 EIP-3009 authorization on X Layer
> with the wallet's TEE key, replays the request with the
> `PAYMENT-SIGNATURE` header, and returns the final JSON. That skill's
> internal CLI surface is not something this guide drives directly.

---

## Prerequisites Check (run once)

1. **Agent payments protocol available** — the `okx-agent-payments-protocol`
   skill must be loadable (ships with `npx skills add okx/onchainos-skills`).
   You can confirm the toolchain is present:
   ```bash
   onchainos --version
   ```
   Install via `npx skills add okx/onchainos-skills` if absent. (This
   guide never invokes a payment subcommand directly — the payments
   protocol skill owns that.)

2. **Wallet has USD₮0 on X Layer** (checked via the agentic-wallet skill):
   ```bash
   onchainos wallet balance --chain xlayer --token-address 0x779ded0c9e1022225f8e0630b35a9b54be713736
   ```

   If the balance is below `0.001`, bridge or fund:
   - **Bridge USDT from another chain** via `okx-dex-bridge`. Source: BSC USDT / Ethereum USDT / Base USDT etc.; destination: X Layer.
   - **Withdraw USDT from OKX exchange** with X Layer as the destination network. The wallet on the receiving end auto-wraps into USD₮0 (contract `0x779ded0c…`).

   ⚠️ The token on X Layer **must be USD₮0** at the exact contract above. Other USDT representations (Stargate-bridged, Wormhole-bridged, etc.) are NOT accepted by Twin3's payment route.

3. **(Optional, developer sanity check) Confirm the challenge endpoint is alive**:
   ```bash
   curl -i "https://human.twin3.ai/v1/human?wallet=0x344659f3eF3c2D2A0CDF071Ea13fa87867777777" | head -5
   ```
   Expected: `HTTP/2 402` with a `payment-required:` header. This is a
   raw curl probe for humans debugging — it is NOT how the agent pays.

---

## Path A — Single-wallet verification

User: "Is `0xVITALIK_OR_WHATEVER` a real person on Twin3?"

### Step 1. Issue the request

```
GET https://human.twin3.ai/v1/human?wallet=0xVITALIK_OR_WHATEVER
```

The first response is `HTTP 402`. The **OKX Agent Payments Protocol**
picks X Layer's USD₮0 entry from the challenge's `accepts` array, signs
an EIP-3009 authorization with the wallet's TEE key, retries the request
with the `PAYMENT-SIGNATURE` header, and yields the final JSON. You do
not assemble or invoke any of that — it happens because the 402
triggers the payments-protocol skill.

### Step 2. Interpret the response

Typical response:

```json
{
  "wallet": "0xvitalik...",
  "isHuman": true,
  "score": 105,
  "scoreFloor": 15,
  "scoreSource": "firestore",
  "sbtId": 142031,
  "queriedAt": "2026-05-11T..."
}
```

Tell the user (example phrasings):

- `isHuman: true, score: 105` → "Yes, this wallet has cleared reCAPTCHA + Google + Apple ID + 2FA verifications via Twin3 (humanity score 105/255)."
- `isHuman: false, sbtId: null` → "No, this wallet hasn't claimed a Twin3 humanity SBT. It's either a fresh wallet, never went through Twin3 onboarding, or is genuinely a non-human."

DO NOT speculate beyond what the score tells you. The score is a verification depth metric, not a "trustworthiness" or "value" judgement.

---

## Path B — Threshold gate for an onboarding flow

User: "Block new signups from wallets below humanity score 45 (= at least reCAPTCHA + Google OAuth)."

### Setup

The agent should add this gate as a precondition for whatever signup action follows. Inside the precondition, issue:

```
GET https://human.twin3.ai/v1/human?wallet=<NEW_USER_WALLET>&min_score=45
```

(402 → auto-paid by the **OKX Agent Payments Protocol** → 200 with `passes`.)

### Decision

- `passes: true` → proceed with signup.
- `passes: false` → reject. Suggested user-facing message: "We require Twin3 humanity score ≥ 45 (reCAPTCHA + Google OAuth) to register. Visit twin3.ai to complete verification, then try again."

---

## Path C — Per-method compliance check

User: "We require 2FA AND Apple ID verifications to enter the premium tier. Check `0xWHATEVER`."

### Issue

```
GET https://human.twin3.ai/v1/human?wallet=0xWHATEVER&check=apple,g2fa
```

(Same payment flow — plain GET, 402 auto-paid.)

### Response

```json
{
  ...
  "verifications": {
    "apple": true,
    "g2fa": false
  }
}
```

### Decision

Both must be true to grant premium. Report each method's status individually so the user knows what's missing.

---

## Path D — Batch sybil filter for an airdrop

User: "I'm dropping a 200-wallet airdrop list. Filter to only humans at score ≥ 70."

### Step 1. Confirm cost

200 × $0.001 = $0.20 USD₮0. Ask the user to approve before kicking off (rule: always confirm batch costs above ~$0.10).

### Step 2. Loop the threshold query

For each wallet in the list, issue one request. Each iteration is an
independent x402 settlement (no batch endpoint, no bulk discount):

```
for w in wallets.txt:
    resp = GET https://human.twin3.ai/v1/human?wallet={w}&min_score=70
           # 402 → OKX Agent Payments Protocol pays $0.001 USD₮0 → 200
    if resp.passes == true:
        append w to qualified.txt
```

The agent issues each GET through its normal HTTP capability; the 402
on each call is settled by the payments-protocol skill. Stay below ~10
concurrent in-flight requests to avoid the upstream rate limiter.

### Step 3. Report

Return `qualified.txt` to the user with a summary:

> Filtered 200 wallets at threshold 70:
> - 142 qualified (humans above score 70)
> - 58 filtered out (below 70 or no Twin3 verification)
> - Total cost: $0.200 USD₮0
> - Estimated sybil savings (if the airdrop value is $X/wallet): 58 × $X

---

## Error Recovery

| Symptom | Likely cause | Fix |
|---|---|---|
| Payment fails: "insufficient balance" reported by the **OKX Agent Payments Protocol** | Wallet has < 0.001 USD₮0 on X Layer | Bridge or fund (see Prerequisites) |
| Server returns `invalid_wallet` | Address malformed | Validate `^0x[a-fA-F0-9]{40}$` before retrying |
| Server returns `invalid_check` | Method name typo | Valid methods: recaptcha, google, discord, line, telegram, apple, g2fa, onchain, worldid |
| Server returns 500 `internal_error` | Backend hiccup (BNB Chain RPC or Firestore) | Retry in 10s; if persistent, exclude that wallet from batch filters |
| Payments protocol reports a settle error / `{"code":-1,"msg":"unknown error"}` | Payment route mismatch (very rare) | Confirm the funded asset is USD₮0 (`0x779ded0c…`) on X Layer, not USDC; the 402 `accepts` array advertises both Base USDC and X Layer USD₮0 |

---

## Cost Cheatsheet

| Scenario | Approx cost |
|---|---|
| Single wallet verification | $0.001 |
| Single threshold check | $0.001 |
| Per-method check (any number of methods named) | $0.001 (the price doesn't scale with method count) |
| 100-wallet sybil filter | $0.10 |
| 1,000-wallet sybil filter | $1.00 |
| 10,000-wallet sybil filter | $10.00 |

X Layer broker gas is sponsored — buyer pays only the $0.001 token transfer per call.
