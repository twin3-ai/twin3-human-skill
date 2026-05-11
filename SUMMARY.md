## Overview

Twin3 Human is a humanity-verification API that answers "is this wallet a real person?" for any EVM address, backed by 141,871 Twin Matrix Soulbound Tokens on BNB Chain plus the canonical Twin3 identity registry. Each query costs $0.001 USD₮0 on X Layer (zero gas via OKX Onchain OS x402 broker).

Core operations:

- **Verify a wallet's humanity score** — returns isHuman + score 0-255 + sbtId in a single paid call
- **Threshold-pass check** — answer "is this wallet above humanity score N?" with a yes/no `passes` verdict for any threshold N
- **Per-method verification check (ZK-style)** — buyer names specific verifications to check (reCAPTCHA, Google, Apple, Telegram, 2FA, etc.); we return verdicts only for the methods asked, never the full set
- **Batch sybil filter** — loop the per-wallet query across an airdrop list to drop bots and pre-launch sybil farmers

## Prerequisites

- `onchainos` CLI installed (`npx skills add okx/onchainos-skills`)
- An Onchain OS Agentic Wallet (or any OKX-compatible EVM signer) with USD₮0 on X Layer (chain id 196). Minimum balance: $0.001 USD₮0 per query.
- Bridge USDT to X Layer if needed via `okx-dex-bridge` or OKX exchange withdrawal (select X Layer as the destination network).

## Quick Start

1. **Check your USD₮0 balance on X Layer**:

   ```
   onchainos wallet balance --chain xlayer --token 0x779ded0c9e1022225f8e0630b35a9b54be713736
   ```

2. **Verify a single wallet** (replace `<WALLET>` with the EVM address to check):

   ```
   onchainos payment pay --url "https://human.twin3.ai/v1/human?wallet=<WALLET>"
   ```

   Reads back JSON with `isHuman`, `score` (0-255), `sbtId`, `scoreSource`. The agent paid $0.001 USD₮0 from your wallet to Twin3's receiver in the same flow.

3. **Threshold filter** (for airdrop / community gating — example: keep only wallets at score ≥ 70):

   ```
   onchainos payment pay --url "https://human.twin3.ai/v1/human?wallet=<WALLET>&min_score=70"
   ```

   Adds `passes: true|false` to the response. Loop across your wallet list to filter.

4. **Per-method check** (ZK-style — name only the methods you want to know about):

   ```
   onchainos payment pay --url "https://human.twin3.ai/v1/human?wallet=<WALLET>&check=telegram,apple,g2fa"
   ```

   Returns `verifications: {telegram: bool, apple: bool, g2fa: bool}` — only the methods you named. The full verification list never leaves Twin3.

For full command reference, response shapes, error tables, and batch-filter pseudocode, see `SKILL.md`.

## Score Reference

| Score | Verification stack the wallet has cleared |
|---|---|
| 0 | No Twin3 verification |
| 15 | reCAPTCHA (baseline anti-bot) |
| 45 | + Google OAuth |
| 70 | + Apple ID |
| 105 | + 2FA |
| 255 | Full verification stack (all methods including g2fa, onchain proof, optional WorldID) |

Higher score = more verification methods cleared by the wallet's owner. Recommended thresholds:

- **Bot filter** for free signups / mints: `min_score=15`
- **Airdrop sybil filter**: `min_score=70`
- **High-value financial gates / governance / DAO membership**: `min_score=105` or `min_score=150`
