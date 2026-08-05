# Showcase post — `#solana-bounty`

Draft for the Discord showcase. Paste as one message; Discord renders the
formatting. Keep the video attached rather than linked if the file size allows.

---

**Baixa — a USDC invoice agent that holds no keys**

Built on ZeroClaw + Solana for the Superteam Brasil bounty. Tier 1: stock release
binary, zero plugins, zero WASM. Skills, SOPs, config, and the built-in
`http_request` tool.

It lives in Telegram. You type `invoice Acme 250 USDC for August`, it issues a
Solana Pay link and QR, then polls the chain every two minutes and closes the
invoice when the payment verifies.

**Custody tier T1.** It never holds a private key and never signs. It builds the
payment URL; the payer signs in their own wallet. A total compromise of the agent
costs you one wrong invoice, not your balance.

**Status changes come from exactly one place.** Step 4 of the reconcile SOP,
after four conditions all hold: no on-chain error, the mint matches, the
destination *owner* matches (read from `postTokenBalances`, not the token account
address), and the amount is at least the invoice. No chat message can set a
status, because nothing in the system has a tool that writes one.

Four states, not two: `open`, `partial`, `paid`, `flagged`. A `destination
mismatch` parks the run on a human checkpoint the agent cannot clear itself.

---

**What is and is not demonstrated**

Invoice issue is real and verified from the runtime trace rather than the reply:
the recipient in the emitted `solana:` URL matches the config constant character
for character, and the reference decodes to a valid 32-byte pubkey unseen on
chain. Reconcile runs all seven steps against a live open invoice, queries the
chain, finds nothing, and leaves the status untouched.

**No real payment has been reconciled.** Steps 3 to 5 have never seen a matching
transaction. Two of five injection vectors have live transcripts; the other three
are labelled predictions, not results. The provider balance ran out, and a free
provider turned out to be unable to drive this agent at all — that became finding
13. All of this is stated in the README rather than left for you to discover.

**The part I'd actually like reviewed**

The agent working is the easy half. The repository documents **thirteen defects**
in the platform and in my own configuration, each with the source line that
explains it. Every one was found by running it, not reading about it. Each passed
`zeroclaw doctor`, `sop validate` and `skills list` while broken, and several
produced no error at any log level.

Four that cost the most:

- **A SOP cron trigger fires and nothing executes it.** The daemon hands the
  dispatch result to a function that only logs. Only `deterministic` runs are
  driven to a terminal state; everything else holds its concurrency slot
  forever, so the SOP fires once and every later tick is skipped with
  `cooldown or concurrency limit reached`.

- **A skill can be installed, listed, and never reach the agent.** The bundle
  `include` list matches the skill's `name:` frontmatter, not its directory. A
  hyphen against an underscore filtered my invoice skill out of every turn,
  silently. `zeroclaw skills list` does not apply that filter, so it reported
  the skill as installed the entire time it was invisible.

- **No free tier can run this agent, and the reason is arithmetic.** Groq caps
  at 12,000 tokens per minute. This agent's system prompt is ~13,000 tokens per
  call, every skill body inlined at `prompt_injection_mode = "full"` — measured,
  not estimated. One turn does not fit in the per-minute quota. Any free tier
  below roughly 15k TPM is out for a Tier 1 agent that inlines its skills.

- **The ledger disappears from its own memory store.** The SOP audit logger
  writes each step, including its tool calls *and their outputs*, into the same
  memory the ledger lives in. A `memory_recall` for `baixa_ledger` therefore
  becomes searchable text containing `baixa_ledger` and outranks the real
  record. Reconciliation then reports success while silently checking nothing.

Three of the thirteen are corrections to my own earlier claims. They're in the
commit history as corrections, with the wrong diagnosis still visible above the
right one.

---

## Who it's for**

Freelancers and small studios who bill in USDC and reconcile by hand: issue an invoice, then keep checking solscan to see whether it landed, whether the amount was right, whether it went to the right address. That check is mechanical, easy to get wrong when tired, and exactly what nobody wants to hand a bot that holds keys. Baixa does the checking and holds nothing. Brazil-relevant in the obvious way: USDC receivable, BRL invoice, and the reconciliation step is the part that actually costs time.

## Why there's no bot link to try

`[peer_groups.baixa_operator]` authorizes exactly one numeric Telegram ID. Every other sender is ignored, so a public link would either do nothing or, if I opened it up, contradict the threat model rather than demonstrate it. Reproduce it instead — SETUP.md is written for that, and every trap I hit on the way is a numbered section in it rather than a footnote.

## ZeroClaw features used

Skills, the SOP engine with per-step tool scopes and an out-of-band approval checkpoint, declarative `[cron]` agent jobs, the Telegram channel with peer-group authorization, sqlite-backed memory, and the built-in `http_request` tool. Nothing was built: no plugins, no WASM, no forks. If a capability isn't in that list, this agent doesn't have it.

---

**Repository:** https://github.com/bodyahhellatrigger/baixa

Start with `SUBMISSION.md` for the findings, `THREAT_MODEL.md` for what is
structural versus behavioural versus not claimed, and `INJECTION_TEST.md` for
five attacks with the expected result written before the run and the live
transcript after.

Happy to be told where the threat model is wrong. That's more useful to me than
being told it looks solid.
