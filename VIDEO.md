# Video script — 3:00 max

Working notes for the demo recording. Not part of the submission's argument;
delete before final push if it reads as clutter.

## Before you press record

- [ ] Daemon stopped. Config on `claude-haiku-4-5`. Cron **paused**.
- [ ] Ledger cleared: `zeroclaw memory clear --category core --yes`
- [ ] Audit noise cleared: `zeroclaw memory clear --category sop --yes`
- [ ] Sending wallet funded: 1 USDC plus ~0.01 SOL for the fee and the ATA
- [ ] Telegram open on a clean chat with the bot, `/` menu visible
- [ ] Terminal ready with `git log --oneline` and the repo open in an editor
- [ ] Screen recording at 1080p, system audio off, no notifications

Start the daemon and resume cron **immediately before** the first take. Reconcile
fires every two minutes; a payment lands inside one or two cycles.

---

## 0:00–0:20 — The claim

Screen: the Telegram chat, empty.

> "This is Baixa. It issues USDC invoices on Solana, watches the chain, and
> closes an invoice when the payment actually verifies.
>
> It holds no private keys and it never signs. It builds a payment link; the
> payer signs in their own wallet. If someone fully compromises this agent, the
> worst they get is a wrong invoice. They cannot move a cent."

Do not say "secure". Say what it cannot do.

---

## 0:20–1:10 — Issue

Screen: type into Telegram.

```
invoice Acme Studio 1 USDC for the demo
```

Reply arrives with the invoice number, amount, truncated reference, `solana:`
URL and QR.

> "The recipient address in that link did not come from my message. I never
> typed it. It comes from operator config, and the skill compares the URL it
> built against that constant character for character before it writes anything.
> If they disagree it refuses and writes nothing."

Cut to terminal, show the trace line proving it:

```sh
# recipient in the emitted URL vs the config constant
```

Then press `/` in Telegram to show the menu, and run `/status`.

> "Three commands. Issue, list, status. The status line at the bottom says
> whether anything needs a human."

---

## 1:10–2:00 — Pay, and let it verify

Screen: scan the QR with the paying wallet, send 1 USDC.

While it confirms, talk over it:

> "Four conditions have to hold before this closes. No on-chain error. The token
> is the configured USDC mint. The destination *owner* is the configured wallet,
> read from post-token-balances, not the token account address. And the amount
> is at least the invoice.
>
> All four, or the status does not move."

Wait for the reconcile cycle. Show the `✅ Invoice #1 — 1 USDC paid` message
with the solscan link. Click it. Show the transaction on solscan.

> "That is the only way a status changes. There is no tool that sets one. I
> asked it in chat to mark an invoice paid; the answer is in the repository
> along with the ledger before and after."

---

## 2:00–2:40 — What actually took the time

Screen: the repository, `SUBMISSION.md` scrolled to *Things that cost time*.

> "The agent working is the easy half. Twelve defects are written down here,
> each with the source line that explains it. Every one was found by running it,
> not by reading about it. Each passed `doctor`, `sop validate` and `skills
> list` while broken."

Pick **two**, on screen, ten seconds each. Suggested pair:

1. **The skill that was never in the prompt.** The bundle include list matches
   the skill's `name:` frontmatter, not its directory. A hyphen against an
   underscore. `skills list` does not apply that filter, so it reported the
   skill installed the whole time it was invisible.

2. **The ledger that disappears from its own store.** The SOP audit logger
   writes each step *and its tool outputs* into the same memory as the ledger.
   A recall for `baixa_ledger` becomes text containing `baixa_ledger`, outranks
   the real record, and reconciliation reports success while checking nothing.

> "Three of the twelve are corrections to my own earlier claims. They are in the
> history as corrections, with the wrong diagnosis still visible above the right
> one."

---

## 2:40–3:00 — What is not claimed

Screen: `THREAT_MODEL.md`, section 9.

> "Invoice creation is model-mediated. A strong enough injection could put a
> wrong address in one invoice. Blast radius is one invoice, flagged within two
> minutes, because the agent holds no keys.
>
> The reference is model-generated, not from a CSPRNG. Raw RPC JSON does reach
> the model context for exactly one step, and at Tier 1 there is no interception
> point without a plugin.
>
> All of that is written down. None of it touches custody."

End on the repository URL.

---

## Rules for the take

- **No adjectives about the software.** Show the behaviour and let it be judged.
- **Never claim the payment worked before it lands on screen.** If the cycle is
  slow, wait in silence rather than narrating a result that has not arrived.
- **If something breaks on camera, keep it and say what happened.** A recovered
  failure is more convincing than a clean run, and this project's argument is
  that the failures are where the work is.
- Do not read the findings verbatim. Point at them and let the viewer read.

## Fallback if the payment does not confirm in time

Do not fake it and do not cut around it. Say so:

> "That has not confirmed yet. The ledger still says open, which is the correct
> behaviour — it does not guess."

Then show a previously recorded confirmation, stated as such.
