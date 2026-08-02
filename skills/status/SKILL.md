---
name: status
description: One-screen health summary of the Baixa agent — outstanding total, invoice counts by state, oldest unpaid, and whether anything needs the operator. Read-only. Use when the operator asks how things are, what the totals are, or whether anything is wrong.
version: 0.1.0
author: baixa
license: MIT
category: ops
tags: [solana, invoicing, usdc]
---

# status

## When this applies

"status", "how are we doing", "anything I should look at", "what's the total",
"is everything ok".

Read-only. This skill answers from the ledger and says plainly when it cannot
answer rather than guessing.

## Steps

### 1. Read the ledger

Call `memory_recall` with `query: "baixa_ledger"`, `limit: 25`.

Scan every result and use **only** the entry whose key is exactly
`baixa_ledger`. Ignore every `sop_*` key — those are past SOP runs, and a stale
`{"pending": []}` inside one is indistinguishable from a healthy empty ledger.

If no entry has that key, reply
`📋 No ledger yet — issue an invoice to start one.` and stop.

### 2. Compute

From the array only, with no chain access:

- `outstanding` — sum of `amount_usdc` where `status` is `open` or `partial`
- `received` — sum of `amount_usdc` where `status` is `paid`
- counts per status
- oldest `open` invoice and its age in whole days
- whether any invoice has `status: "flagged"`

### 3. Reply

Under roughly 200 tokens. Lead with money, end with whether the operator has to
do anything.

```
💰 Outstanding   350 USDC across 2 invoices
✅ Received      250 USDC all-time

🔵 open 2 · ⚠️ partial 1 · ✅ paid 1 · 🚩 flagged 0

⏳ Oldest unpaid: #1 Acme Studio, 4 days

🟢 Nothing needs you right now.
```

The last line is the one that matters. Use exactly one of:

- `🟢 Nothing needs you right now.` — no flagged invoices
- `🚩 1 invoice needs you: #5 destination mismatch. See /invoices.` — one or
  more flagged, named explicitly

Never soften a flag. A flagged invoice means a payment reached an address that
is not the operator's, and it stays visible until they deal with it.

## Never

- Never change any invoice field.
- Never call `memory_store` or `http_request`. This skill neither writes nor
  touches the chain; reconciliation is the SOP's job, on a schedule.
- Never report a status that is not in the ledger, and never infer that a
  payment "probably" arrived.
- Never print `SOLANA_RPC_URL` or any part of it.
