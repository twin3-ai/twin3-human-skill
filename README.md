# twin3-human

> Humanity-verification skill for [OKX OnchainOS](https://web3.okx.com/onchainos) / Agentic Wallet. Answers "is this wallet a real person?" for any EVM address, with per-method ZK-style verification checks. Pay per query with USD₮0 on X Layer (zero gas).

Backed by **141,871 [Twin Matrix](https://twin3.ai) Soulbound Tokens** on BNB Chain plus the canonical Twin3 humanity registry. The full verification list never leaves Twin3 — only verdicts.

---

## Install

```bash
npx skills add twin3-ai/twin3-human-skill
```

Or as a Claude Code plugin (also works in Cursor / Codex / OpenCode):

```bash
# Claude Code — add the marketplace, then install the plugin
/plugin marketplace add twin3-ai/twin3-human-skill
/plugin install twin3-human@twin3
```

`twin3` is the marketplace name declared in
`.claude-plugin/marketplace.json`; `twin3-human` is the plugin/skill
name. Once installed the skill auto-triggers on humanity-verification
intents — no manual invocation needed.

## Quick demo

After install, ask your agent:

> Is `0xd8da6bf26964af9d7eed9e03e53415d37aa96045` a real human on Twin3? Verify it.

The agent will:

1. Detect the humanity-verification intent → load this skill.
2. Build the URL `https://human.twin3.ai/v1/human?wallet=0xd8da6bf26964af9d7eed9e03e53415d37aa96045`.
3. Get HTTP 402 from Twin3 → call `okx-agent-payments-protocol` to sign + pay $0.001 USD₮0 on X Layer.
4. Re-fetch with `PAYMENT-SIGNATURE` → receive `{isHuman, score, sbtId, …}`.
5. Report the verdict to you.

## What this skill does

| Capability | Trigger |
|---|---|
| **Single-wallet verification** | "is 0x… real?", "humanity score of 0x…", "verify 0x…" |
| **Threshold gate** | "is 0x… above score 70?", "filter wallets ≥ score N" |
| **Per-method ZK check** | "did 0x… verify with Telegram and Apple ID?" |
| **Batch sybil filter** | "filter 100 wallets at threshold 70 for airdrop" |

## What this skill does NOT do

- Move funds beyond the $0.001/query micropayment
- Sign anything for the wallet under verification (you only query its public verdict)
- KYC off-chain users (Twin3 is wallet-keyed only)
- Mint Twin3 SBTs (use the Twin3 onboarding flow at twin3.ai for that)

## Files in this plugin

| File | Purpose |
|---|---|
| `SKILL.md` | LLM-facing skill definition, command list, examples |
| `SUMMARY.md` | English overview, prerequisites, quickstart |
| `plugin.yaml` | Plugin manifest (schema, category, tags, payment metadata) |
| `.claude-plugin/plugin.json` | Claude Skill registration |
| `references/api.md` | Full HTTP API reference (params, response, error table) |
| `references/quickstart-okx.md` | End-to-end recipe for Agentic Wallet users |
| `LICENSE` | MIT |

## Payment

| Network | Token | Cost / call | Notes |
|---|---|---|---|
| **X Layer** (recommended) | USD₮0 (`0x779ded0c…`) | $0.001 | Zero gas via OKX Onchain OS x402 broker |
| Base (fallback) | USDC (`0x833589fC…`) | $0.001 | For non-OKX agents; standard Base gas applies |

## Security

- This plugin is **read-only**. It signs only the $0.001 EIP-3009 authorization for the per-query payment; nothing else.
- Private keys never leave the Agentic Wallet TEE. The plugin only constructs URLs and hands them to `okx-agent-payments-protocol` for signing.
- Twin3 never exposes the queried wallet's full verification list — only verdicts for methods the caller specifically asks about (ZK-style).
- Report security issues to wen@twin3.ai or via the [bug bounty](https://twin3.ai/security).

## Author

[Twin3.ai](https://twin3.ai) — Digital Identity Infrastructure. Twin Matrix · 256-dimension human profile · proof-of-humanity for the agent economy.

## License

MIT — see [LICENSE](./LICENSE).
