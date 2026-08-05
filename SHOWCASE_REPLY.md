**Who it's for**

Freelancers and small studios who bill in USDC and reconcile by hand: issue an invoice, then keep checking solscan to see whether it landed, whether the amount was right, whether it went to the right address. That check is mechanical, easy to get wrong when tired, and exactly what nobody wants to hand a bot that holds keys. Baixa does the checking and holds nothing. Brazil-relevant in the obvious way: USDC receivable, BRL invoice, and the reconciliation step is the part that actually costs time.

**Why there's no bot link to try**

`[peer_groups.baixa_operator]` authorizes exactly one numeric Telegram ID. Every other sender is ignored, so a public link would either do nothing or, if I opened it up, contradict the threat model rather than demonstrate it. Reproduce it instead — SETUP.md is written for that, and every trap I hit on the way is a numbered section in it rather than a footnote.

**ZeroClaw features used**

Skills, the SOP engine with per-step tool scopes and an out-of-band approval checkpoint, declarative `[cron]` agent jobs, the Telegram channel with peer-group authorization, sqlite-backed memory, and the built-in `http_request` tool. Nothing was built: no plugins, no WASM, no forks. If a capability isn't in that list, this agent doesn't have it.
