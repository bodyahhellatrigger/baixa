# Video script — 3:00 max

There is no payment demo. The provider balance ran out and a free provider cannot
drive this agent (SUBMISSION.md, finding 13). This script shows what exists and
says plainly what does not.

That is not a consolation shape. The strongest thing this project has is not a
green checkmark; it is thirteen defects found by running a platform that reports
itself healthy while broken. Lead with that.

## Before you press record

- [ ] Repo open at https://github.com/bodyahhellatrigger/baixa
- [ ] Telegram scrolled to the invoice #1 exchange with the QR visible
- [ ] Terminal with `git log --oneline` ready
- [ ] Editor with `SUBMISSION.md` at *Things that cost time*
- [ ] 1080p, no notifications

---

## 0:00–0:25 — What it is and what it cannot do

Screen: the Telegram exchange, invoice #1 with the QR.

> "Baixa issues USDC invoices on Solana from Telegram, then polls the chain and
> closes an invoice when the payment verifies.
>
> It holds no private keys and it never signs. It builds a payment link; the
> payer signs in their own wallet. A total compromise of this agent costs you one
> wrong invoice, not your balance. That constraint is the whole design."

---

## 0:25–1:05 — The part that is demonstrated

Screen: the invoice message, then cut to terminal.

> "This invoice is real. What matters is not that a message appeared — it is
> where the address came from."

Show the check:

```
recipient in URL : 93K1fGpFkaAS9TKiqr49hs9KJZwv9PgnsiDzLkN52niK
RECIPIENT_WALLET : 93K1fGpFkaAS9TKiqr49hs9KJZwv9PgnsiDzLkN52niK
MATCH            : True
```

> "That is read out of the runtime trace, not out of the reply. I never typed
> that address. It comes from operator config, and the skill compares what it
> built against the constant before writing anything.
>
> The reference is 44 characters, valid base58, decodes to exactly 32 bytes, and
> was unseen on chain when issued."

Then show a reconcile run:

```
1: {"pending": [{"id": 1, ... "status": "open"}]}
2: {"candidates": []}
5: {"written": false, "destination_mismatch": false}
   step 6/7 — checkpoint not dispatched
```

> "Seven steps, run against a real open invoice. It queried the chain, found
> nothing, and left the status untouched. Repeatedly. Silence when there is no
> payment is the behaviour you actually want."

---

## 1:05–1:25 — What is not demonstrated

Do not skip this and do not apologise for it.

> "No real payment has been reconciled. Steps three through five have never seen
> a matching transaction. The four-condition check is implemented and readable;
> it is not demonstrated.
>
> Two of five injection vectors were run. The other three are labelled as
> predictions in the repository, not results.
>
> The balance ran out. I tried a free provider and it cannot run this agent —
> that became finding thirteen."

---

## 1:25–2:35 — The part worth the reader's time

Screen: `SUBMISSION.md` at *Things that cost time*.

> "Thirteen defects, each with the source line that explains it. Every one found
> by running the thing. Each passed `doctor`, `sop validate` and `skills list`
> while broken. Several produced no error at any log level."

Pick three, ten seconds each, on screen:

1. **A SOP cron trigger fires and nothing executes it.** The daemon hands the
   result to a function that only logs. Only `deterministic` runs are driven to a
   terminal state. Everything else holds its concurrency slot forever, so the SOP
   fires once and every later tick is skipped.

2. **A skill can be installed, listed, and never reach the agent.** The bundle
   `include` list matches the skill's `name:` frontmatter, not its directory. A
   hyphen against an underscore. `skills list` does not apply that filter, so it
   reported the skill installed the whole time it was invisible. Three wrong
   diagnoses came out of that one character.

3. **The ledger disappears from its own store.** The SOP audit logger writes each
   step *and its tool outputs* into the same memory as the ledger. A recall for
   `baixa_ledger` becomes text containing `baixa_ledger`, outranks the real
   record, and reconciliation reports success while checking nothing.

Cut to `git log --oneline`.

> "Three of the thirteen are corrections to my own earlier claims. They are in
> the history as corrections, with the wrong diagnosis still visible above the
> right one. One of them is a cost ceiling I presented as a control and then made
> inert by switching models without updating the rate sheet. It read one dollar
> while the real bill was eleven."

---

## 2:35–3:00 — Close

Screen: `THREAT_MODEL.md`, section 9.

> "Invoice creation is model-mediated. A strong enough injection could put a
> wrong address in one invoice. Blast radius is one invoice, because the agent
> holds no keys.
>
> The memory store is a trust boundary and I did not treat it as one until it
> bit. Raw RPC JSON reaches the model context for exactly one step, and at Tier 1
> there is no interception point without a plugin.
>
> All of that is written down. None of it touches custody."

End on the repository URL.

---

## Rules for the take

- **Never imply the payment worked.** The sentence "no real payment has been
  reconciled" is in the script for a reason. Say it.
- **No adjectives about the software.** Show behaviour; let it be judged.
- **If something breaks on camera, keep it.** This project's argument is that the
  failures are where the work is. A recovered failure is on-message.
- Do not read findings verbatim. Point at them and let the viewer read.
