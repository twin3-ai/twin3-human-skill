---
name: twin3-human
description: |
  Humanity verification for EVM wallets — answer "is this wallet a real person?" with a humanity score 0-255 plus per-method verification booleans (reCAPTCHA / Google OAuth / Apple ID / Telegram / 2FA / Twin Matrix SBT). Backed by 141,871 Twin Matrix Soulbound Tokens on BNB Chain plus the canonical humanity registry. Pay per query with USD₮0 on X Layer (zero gas via Onchain OS x402 broker).

  Fires on (English): "is this wallet real / a real person / a bot / a sybil", "verify human", "humanity check", "humanity score", "Twin Matrix score", "Twin3 score", "KYA", "Know Your Anonymous", "proof of humanity", "sybil resistance", "sybil filter", "filter bots / sybils from this list", "airdrop sybil filter", "anti-sybil drop", "check if 0x… is a person", "wallet identity score", "is 0x… verified", "does 0x… pass humanity check".

  Fires on (中文): "驗證真人", "驗證真實身份", "人類驗證", "人類分數", "Twin Matrix 分數", "Twin3 分數", "鏈下身份驗證", "錢包是真人嗎", "錢包是不是 bot", "過濾機器人", "過濾女巫", "空投反女巫", "空投篩選真人", "證明真人", "sybil 過濾", "防 bot 驗證".

  Fires on (wallet + verb): an EVM address followed by humanity / identity / verify / sybil / bot verbs in either language. Example: "0x344659… is real?" / "check 0xabc…123 humanity" / "0x… 是真人嗎？".

  Anti-triggers (do NOT fire here, route elsewhere): wallet balance lookup (use okx-wallet-portfolio), token price (okx-dex-market), DApp-named verification (route to okx-dapp-discovery), generic KYC of off-chain users (Twin3 is wallet-keyed only).
license: MIT
metadata:
  author: Twin3.ai
  version: "0.1.0"
  homepage: "https://twin3.ai"
  payment_protocol: "x402"
  payment_network: "eip155:196"
  payment_asset: "USD₮0"
  payment_amount_usd: "0.001"
---

# Twin3 Human — Humanity Verification for AI Agents

## Overview

This skill enables the AI agent to answer **"is this wallet backed by a real person?"** for any EVM address, returning:

1. `isHuman` — boolean, whether the wallet is associated with a verified human identity in the Twin3 registry.
2. `score` — integer 0-255, the cumulative humanity weight of all verifications the wallet's owner has cleared. Reference levels: 15 = baseline reCAPTCHA, 45 = + Google OAuth, 70 = + Apple ID, 105 = + 2FA, 255 = full verification stack.
3. `sbtId` — the wallet's Twin Matrix Soulbound Token id on BNB Chain (immutable, non-transferable). Null if no SBT.
4. `verifications` (optional) — per-method boolean map. Buyer must NAME the methods to check (ZK-style: we never enumerate the full set, only verdicts for methods you specifically ask about). Valid methods: `recaptcha`, `google`, `discord`, `line`, `telegram`, `apple`, `g2fa`, `onchain`, `worldid`.
5. `passes` (optional) — when buyer passes `min_score=N`, a yes/no verdict against that threshold.

Backed by **141,871 Twin Matrix Soulbound Tokens** on BNB Chain plus the canonical humanity registry off-chain. The full verification list never leaves Twin3 — only verdicts go out.

## Pre-flight Checks

Before using this skill, ensure:

1. The `onchainos` CLI is installed and configured (`npx skills add okx/onchainos-skills`).
2. The user's wallet has at least 0.001 USD₮0 on X Layer (chain id 196) — every query costs $0.001. Reference: [okx-wallet-portfolio](../okx-wallet-portfolio/SKILL.md) for balance check, [okx-dex-bridge](../okx-dex-bridge/SKILL.md) for bridging USDT to X Layer if needed.
3. The wallet to be verified is a valid EVM address (`^0x[a-fA-F0-9]{40}$`).

## Endpoint

| Property | Value |
|---|---|
| URL | `https://human.twin3.ai/v1/human` |
| Method | `GET` |
| Payment | `$0.001` USD₮0 on X Layer (`eip155:196`) — single charge per query via x402 / OKX Agent Payments Protocol |
| Price-equivalent on other chains | $0.001 USDC on Base (`eip155:8453`) — for cross-chain agents; X Layer is the canonical zero-gas path |

## Query Parameters

| Param | Required | Format | What it adds |
|---|---|---|---|
| `wallet` | yes | `0x` + 40 hex | The EVM address to verify |
| `min_score` | no | int 0-255 | Adds `passes: bool` to the response — yes if `score ≥ min_score` |
| `check` | no | CSV of method names | Adds `verifications: {method: bool}` for ONLY the methods you name. Valid: `recaptcha,google,discord,line,telegram,apple,g2fa,onchain,worldid` |
| `include` | no | CSV of richer fields | `verified_at` adds SBT mint timestamp; `dimensions_filled` adds non-zero Twin Matrix dimension count |

## Commands

### Verify a single wallet (baseline)

**When to use**: User asks "is 0x… a real person?", "verify this wallet", "humanity score of 0x…", "錢包 0x… 是真人嗎？"

```bash
# Detect the 402 challenge with the x402 / Onchain OS Agent Payments Protocol
onchainos payment pay \
  --url "https://human.twin3.ai/v1/human?wallet=<WALLET>"
```

**Output** (example):

```json
{
  "wallet": "0x344659f3eF3c2D2A0CDF071Ea13fa87867777777",
  "isHuman": true,
  "score": 105,
  "scoreFloor": 15,
  "scoreSource": "firestore",
  "sbtId": 142031,
  "queriedAt": "2026-05-11T14:51:06.979Z"
}
```

**How to interpret**:
- `isHuman: true` + `score >= 15` → the wallet is a verified human. Higher score = stronger verification.
- `isHuman: false` or `sbtId: null` → no Twin3 verification for this wallet. Either not a Twin3 user, or never claimed.

### Threshold-pass check

**When to use**: User says "is 0x… above humanity threshold 70?", "篩選分數 ≥ 70 的", "filter wallets above bot-floor".

```bash
onchainos payment pay \
  --url "https://human.twin3.ai/v1/human?wallet=<WALLET>&min_score=<N>"
```

**Output** adds:
```json
"passes": true
```

### Per-method verification check (ZK-style)

**When to use**: User asks "did 0x… verify via Telegram AND Apple ID?", "驗證 0x… 是否做過 2FA + Google OAuth".

```bash
onchainos payment pay \
  --url "https://human.twin3.ai/v1/human?wallet=<WALLET>&check=telegram,apple,g2fa"
```

**Output** adds:
```json
"verifications": {
  "telegram": true,
  "apple": false,
  "g2fa": true
}
```

**Important**: The response is ZK-style — only the methods the buyer NAMES are answered. We never enumerate the full set. Buyer must already know which methods to ask about.

### Batch sybil filter (airdrop use case)

**When to use**: User has a list of wallets and wants to keep only the ones above a humanity threshold. Example: "filter sybils from this 100-wallet list, drop anyone below 70", "從這 100 個錢包過濾出分數 ≥ 70 的真人".

For each wallet in the list, run a `threshold-pass` query (see above) with `min_score=<threshold>`. Aggregate results. At $0.001 per wallet, a 1,000-wallet filter costs $1.

Pseudo-loop:
```bash
for w in $WALLETS; do
  onchainos payment pay --url "https://human.twin3.ai/v1/human?wallet=$w&min_score=70" \
    | jq -r 'select(.passes==true) | .wallet'
done > qualified-humans.txt
```

The AI agent should ask the user to confirm total cost before kicking off batch — at scale, $1 / 1K wallets is cheap but ask for confirmation if list exceeds 100.

## Examples

### Example 1: Verify a known wallet (single query)

User: "Is `0xd8da6bf26964af9d7eed9e03e53415d37aa96045` (vitalik.eth) a real person on Twin3?"

1. Run `onchainos payment pay --url "https://human.twin3.ai/v1/human?wallet=0xd8da6bf26964af9d7eed9e03e53415d37aa96045"`
2. Read the response. If `isHuman: true`, report the score with the reference levels above. If `false`, note that Vitalik has not claimed a Twin3 humanity SBT (yet).

### Example 2: Filter sybils from an airdrop list

User: "I have 50 wallet addresses for the next airdrop. Filter out anyone below humanity score 70."

1. Confirm the cost: 50 × $0.001 = $0.05 USD₮0.
2. Confirm the user has enough USD₮0 on X Layer (`okx-wallet-portfolio`).
3. Loop the threshold-pass query.
4. Return the qualifying wallets + a count of how many sybils were filtered.

### Example 3: Method-specific compliance check

User: "We require 2FA + Apple ID for this onboarding step. Check if `0xabc…123` has both."

1. Run `onchainos payment pay --url "https://human.twin3.ai/v1/human?wallet=0xabc…123&check=apple,g2fa"`.
2. Read `verifications.apple` and `verifications.g2fa`. Report each method's status to the user.

## Case Studies

These are the agent-builder patterns that have driven the bulk of inbound queries since launch. Use them as templates when proposing this skill to a user who has the same shape of problem.

### CS-1 — Airdrop sybil filter, score-gated

**Shape**: project has a candidate list of 500–50,000 wallets (from a Discord raid, a quest platform, a Galxe campaign). Wants to drop the bottom layer of obvious sybils without paying for full KYC.

**Pattern**:
1. Agent fetches the candidate list from the project's source (CSV, Dune query, snapshot).
2. Loops `threshold-pass` at `min_score=70` (= reCAPTCHA + Google + Apple, the level real users almost always clear and bots rarely do).
3. Returns two files: `qualified.csv` (humans) + `filtered.csv` (with reason).
4. Reports filter rate. Typical sybil ratio observed: 35–55% of raw raid lists, 8–20% of organic community lists.

**Why this beats alternatives**: cheaper than Gitcoin Passport (no per-user signup friction — Twin3's score is wallet-keyed and pre-existing), faster than manual review, leaves a paid audit trail for governance.

### CS-2 — Discord / Telegram community bouncer

**Shape**: community wants verified-human gate before granting `verified` role or chat access. Bot membership runs into the thousands; manual review doesn't scale.

**Pattern**:
1. New member binds wallet via the community's normal flow (sign message).
2. Agent calls this skill with `wallet=<bound>` and `min_score=45` (= at least Google OAuth, weeds out throwaway bots without alienating new users).
3. If `passes: true`, role granted. If `false`, polite redirect to twin3.ai to complete verification, retry-able once verified.

**Operating cost**: a community of 5,000 newcomers/month costs $5/month in Twin3 fees — cheaper than one moderator-hour.

### CS-3 — DePIN / quest reward distribution

**Shape**: protocol distributes ongoing rewards to participants. Reward batch fires daily, monthly, or per-event. Distinguishing real users from farm-rigs is the difference between a healthy economy and a sybil collapse.

**Pattern**:
1. Before each reward batch, run the recipient list through `threshold-pass` at `min_score=105` (= 2FA tier; high-confidence human).
2. Below-threshold wallets get a downgraded reward (e.g. 25%) instead of zero, with an in-app prompt to "verify your humanity at twin3.ai to unlock full rewards" — this converts sybil-suspect wallets into either upgraded real users or self-filtered drop-offs.
3. Audit log retains the per-wallet verdict + the $0.001 paid x402 settlement — sufficient for community governance Q&A ("why was wallet X downweighted?").

**Why per-method check matters here**: protocols with a strict on-chain identity policy can additionally require `check=worldid,onchain` for the top reward tier, layering Worldcoin / ENS-style proofs on top of Twin3's general humanity score.

### CS-4 — Agent-to-agent gate (the meta case)

**Shape**: an autonomous agent (e.g. a trading bot, a DAO governance bot) needs to know whether the wallet *it is paying or interacting with* is a real person — for example before honoring a high-value swap quote, before voting weight aggregation, or before honoring a referral.

**Pattern**:
1. Other agent's logic identifies a wallet of interest (`0xPEER`).
2. Calls `human.twin3.ai/v1/human?wallet=0xPEER&include=verified_at,dimensions_filled`.
3. Routes its own action by the verdict — e.g. higher slippage tolerance and tighter rate limits for `score < 45`, normal behaviour for `score ≥ 70`.

This is the canonical "x402 service composed inside another x402 service" pattern and the reason this skill ships with Onchain OS — agents that have an `okx-agent-payments-protocol` setup already paid for can call this one without any additional onboarding.

## Error Handling

| Error | Cause | Resolution |
|---|---|---|
| HTTP 400 `invalid_wallet` | Wallet param is not a valid `0x…40hex` address | Validate format before retry; ask user to re-supply the address |
| HTTP 400 `invalid_min_score` | `min_score` not an integer 0-255 | Clamp or ask user for a valid threshold |
| HTTP 400 `invalid_check` | One of the `check=` method names is unknown | Valid methods: `recaptcha`, `google`, `discord`, `line`, `telegram`, `apple`, `g2fa`, `onchain`, `worldid` |
| HTTP 402 not paying | Wallet has insufficient USD₮0 on X Layer | Bridge USDT to X Layer via `okx-dex-bridge`, or fund via OKX exchange withdrawal (network = X Layer) |
| HTTP 500 `internal_error` | Backend error (rare); BNB Chain RPC or Firestore upstream issue | Retry after 10s. If persistent, the result for that wallet is unknown — exclude from sybil filters |

## Security Notices

- **Read-only**: this skill only reads humanity verdicts. It does not move funds beyond the $0.001 per-query payment, does not write to chain, and cannot expose private keys.
- **Privacy**: the buyer's wallet address is logged for billing/audit. The QUERIED wallet's address is logged too. Twin3 never exposes the queried wallet's verification list — only the verdicts for the methods the buyer specifically asks about.
- **Payment finality**: payment settles on X Layer in 1-2 seconds via the OKX Onchain OS broker. Each query is one independent settlement; there is no subscription or recurring debit.
- **Trust source**: humanity verdicts come from the canonical Twin3 registry, which aggregates user-opted-in identity verifications. Twin3 does not invent humanity data — every verification was performed by the wallet's owner via the Twin Matrix flow.

## Skill Routing

| Intent | Use skill |
|---|---|
| Just the verification result | **This skill** (`twin3-human`) |
| Wallet's USD₮0 balance on X Layer | `okx-wallet-portfolio` |
| Bridge USDT from another chain to X Layer | `okx-dex-bridge` |
| The actual x402 / Agent Payments Protocol payment mechanism | `okx-agent-payments-protocol` |
| KYC / off-chain identity (not wallet-keyed) | Out of scope — Twin3 only verifies on-chain wallet identities |
| Trading or DeFi action based on humanity score | Route to the trading/DeFi skill of your choice (e.g. `okx-dex-swap`) **after** running this skill for the user filter |

## How This Skill Fits the Stack

```
User intent (natural language)
     │
     ▼
twin3-human SKILL.md      ← (this file) recognizes humanity-verification intent
     │
     ▼
okx-agent-payments-protocol ← handles the x402 / 402 negotiation, payment signing
     │
     ▼
Onchain OS Broker (X Layer)   ← settles the USD₮0 transfer on-chain
     │
     ▼
https://human.twin3.ai/v1/human  ← Twin3 server returns the JSON verdict
     │
     ▼
LLM presents the result to the user
```

The skill is **stateless** from the agent's perspective: every query is a fresh paid HTTP call. The state (humanity scores, SBT registry, Twin Matrix data) lives in Twin3's canonical backend and is updated only when users opt into new verifications via the Twin Matrix flow.
