# Baixa

A USDC invoice reconciliation agent on [ZeroClaw](https://github.com/zeroclaw-labs/zeroclaw) and Solana. It lives in Telegram, issues Solana Pay invoices, polls the chain on a schedule, and closes an invoice only when four on-chain conditions hold.

Built for the Superteam Brasil ZeroClaw bounty. **Tier 1: stock release binary, zero plugins, zero WASM.** Skills, SOPs, config, and the built-in `http_request` tool. Nothing else.

---

## Baixa is a ledger, not a till

It holds no private keys and it never signs. It constructs payment URLs; the payer signs in their own wallet. The worst outcome of a total compromise is a wrong invoice, not drained funds.

**Custody tier T1.** Secrets held: a Telegram bot token, an LLM provider key, a Solana RPC key. None of them can move money.

That constraint is the design, not a limitation apologised for. Everything below follows from it.

---

## Four states, not two

Most invoice bots have `unpaid` and `paid`. Real payments are messier:

| State | Reached when |
|---|---|
| `open` | Issued, nothing on chain yet |
| `partial` | Right token, right destination, amount short |
| `paid` | All four conditions hold |
| `flagged` | Wrong mint, wrong destination, or a duplicate payment |

Status changes only in step 4 of `sops/reconcile/SOP.md`, and only from a `getTransaction` response. No chat message, invoice description, or memo field can set it, hint at it, or justify it.

The four conditions, all of which must hold:

1. `meta.err` is null
2. An SPL transfer whose `mint` equals the configured USDC mint exactly
3. The destination **owner** equals the configured recipient exactly, read from `meta.postTokenBalances[].owner` and never from the token-account address
4. The transferred amount is at least the invoice amount, compared as decimals

A `destination mismatch` is the one outcome that parks the run on a human checkpoint. It cannot be cleared by the agent: `approval_mode = "out_of_band_required"` turns the agent's own `sop_approve` into a no-op.

---

## Status: what is proven and what is not

Honesty here is deliberate. A reader should be able to tell demonstration from claim.

**Verified against a live mainnet deployment:**

- Invoice issue end to end. The recipient and mint in the emitted `solana:` URL match the config constants character for character, verified from the runtime trace rather than from the reply text.
- Reference minting: 44 characters, valid base58, decodes to exactly 32 bytes, unseen on chain.
- Reconcile completing all seven steps against a real open invoice, finding no payment, and leaving `status`, `paid_at`, and `tx_signature` untouched across repeated cycles.
- The step 6 routing guard: with `destination_mismatch` false, the run ends at 6/7 and the human checkpoint is never dispatched.
- Injection vectors A and B, with live transcripts in [INJECTION_TEST.md](INJECTION_TEST.md).

**Not proven, and not claimed anywhere in this repository:**

- **No real payment has been reconciled.** Steps 2 through 7 have run against an empty result set, never against a matching transaction. The four-condition check is implemented and read, not demonstrated.
- Injection vectors C, D and D2 are incomplete. A and B have live transcripts; the rest do not.
- `daily_digest` has never executed.
- `/invoices` and `/status` are written and installed but were never invoked.

The reason is mundane and worth stating rather than hiding: the provider balance ran out mid-session, and topping it up further was not worth it for this submission. A free provider was tried and cannot run this agent — that attempt became finding 13.

Everything above the line was measured from the runtime trace and the ledger, not read off a chat reply. Everything below it is absent. There is no third category.

---

## What this project is actually worth reading for

The agent works. So will most submissions. The part that took the time is documented in [SUBMISSION.md](SUBMISSION.md) under *Things that cost time*: thirteen defects in the platform and in this repository's own configuration, each with a source citation.

Every one of them was found by **running the thing**. Each passed `zeroclaw doctor`, `sop validate`, and `skills list` while broken, and several produced no error at any log level.

A sample:

- **A SOP cron trigger fires and nothing executes it.** The daemon hands the result to a function that only logs. Only `deterministic` runs are driven to a terminal state; everything else holds its concurrency slot forever, so the SOP fires exactly once and every later tick is skipped.
- **A skill can be installed, listed, and never reach the agent.** The bundle `include` list matches the skill's `name:` frontmatter, not its directory. A hyphen against an underscore filtered `create_invoice` out of every turn, silently. `zeroclaw skills list` does not apply that filter, so it reported the skill as installed the whole time.
- **A classifier decides whether your message deserves a reply.** An explicit invoice request was classified as not needing one. The agent loop never ran. Nothing downstream had a chance to fail loudly.
- **The cost ceiling only covers models named in the rate sheet.** A model absent from `[cost.rates.*]` is priced at $0.00, so `enforcement.mode = "block"` never fires. Measured: a $3.00 daily cap read $1.04 while the real bill was $11.44.
- **No free provider can run this agent, and the reason is arithmetic.** Groq's free tier caps at 12,000 tokens per minute. This agent's system prompt is ~13,000 tokens per call, every skill body inlined. One turn does not fit in the per-minute quota.
- **The ledger disappears from its own store.** The SOP audit logger writes each step, including its tool calls *and their outputs*, into the same memory the ledger lives in. A `memory_recall` for `baixa_ledger` therefore becomes searchable text containing `baixa_ledger`, outranking the real record. Reconciliation then reports success while silently checking nothing.

Three of those thirteen are corrections to this repository's own earlier claims. They are written down as corrections rather than quietly fixed, and the commit history shows both the wrong diagnosis and the right one.

---

## Verify it in two minutes

Without installing anything:

1. [SUBMISSION.md](SUBMISSION.md) — the thesis, the failure-path matrix, and the thirteen findings with source lines
2. [THREAT_MODEL.md](THREAT_MODEL.md) — what is structural, what is behavioural, and what is not claimed
3. [INJECTION_TEST.md](INJECTION_TEST.md) — five attacks, each with the expected result written before the run and the live transcript after
4. `git log` — the messages carry the reasoning, including three reversed diagnoses

To run it: [SETUP.md](SETUP.md). It includes the traps above as numbered sections, so a reader reproducing this does not lose the same day.

---

## Layout

```
config.example.toml     annotated; every key cited to the ZeroClaw source
skills/create_invoice/  issue an invoice, build the URL from constants only
skills/invoices/        read-only ledger listing
skills/status/          read-only health summary
sops/reconcile/         seven steps, four conditions, one human checkpoint
sops/daily_digest/      read-only daily summary, no chain access
SPEC.md                 Stage 1 recon: every platform claim with file:line
```

---

## Requirements

ZeroClaw 0.8.3, a Solana RPC endpoint, a Telegram bot token, and an LLM provider key.

One constraint worth knowing before you start: ZeroClaw 0.8.3's agent loop emits a prefill on the last assistant turn, which every Claude model from the 4.6 generation onward rejects with a 400. The 4.5 generation is the newest that this runtime can drive.

---

## Licence

MIT.
