# daily_digest

Once a day, send the operator a single summary of the Baixa ledger. Read-only:
this SOP never changes an invoice status, never touches the chain, and never
writes to memory.

## OPERATOR CONSTANTS

Kept here for cross-checking only. This SOP issues no invoices and verifies no
transactions, so it reads these values but never acts on them. They must match
the copies in `config.toml` and `skills/create_invoice/SKILL.md`.

```
RECIPIENT_WALLET = <RECIPIENT_SOLANA_ADDRESS>
USDC_MINT        = <USDC_MINT_ADDRESS>
```

## Steps

1. **Read the ledger** — One recall, no chain access.
   - tools: memory_recall
   - Call `memory_recall` with `query: "baixa_ledger"`, `limit: 1`, and parse
     the JSON array. An empty or missing ledger means nothing to report: send
     `📋 Baixa — ledger empty` and finish.
   - output: {"type":"object","required":["invoices"],"properties":{"invoices":{"type":"array"}}}
   - on_failure: fail

2. **Build the four sections** — Arithmetic only, no tool calls.
   - **Paid today** — entries whose `status` is `paid` and whose `paid_at`
     falls on today's UTC date. List id, counterparty, amount. Sum the amounts
     into a today total.
   - **Still open** — entries whose `status` is `open` or `partial`. For each,
     compute age in whole days from `created_at` to now. Sort oldest first, so
     the most overdue sits at the top. For `partial` entries show what arrived
     and what is missing.
   - **Flagged** — entries whose `status` is `flagged`. Show the id and the
     recorded reason verbatim. Never summarize a reason away; a
     `destination mismatch` must read as `destination mismatch`.
   - **Running total received** — sum `amount_usdc` across every entry whose
     `status` is `paid`, for all time.
   - output: {"type":"object","required":["sections"],"properties":{"sections":{"type":"object"}}}
   - on_failure: fail

3. **Send the digest** — One message, compact.
   - tools: send_message_to_peer
   - Send to `channel: "telegram.home"`, `target: <operator peer id>`. Keep it
     under roughly 200 tokens. Drop any section that is empty rather than
     printing a header with nothing under it. Shape:

     ```
     📋 Baixa — 2026-08-02

     ✅ Paid today · 250 USDC
     #7 Acme Studio 250

     ⏳ Open · 2
     #9 Foo Ltd 100 (partial: 40 in, 60 short) · 3d
     #12 Bar GmbH 500 · 1d

     🚩 Flagged · 1
     #11 destination mismatch

     Σ received all-time: 4 310 USDC
     ```
   - When every section is empty, send `📋 Baixa — nothing open, nothing paid
     today` rather than staying silent. Unlike reconcile, a daily digest that
     goes quiet is indistinguishable from a broken scheduler, so this one always
     reports.
   - on_failure: fail
