# Baixa — USDC invoice reconciliation on ZeroClaw + Solana

An agent that lives in Telegram, issues Solana Pay invoices, polls the chain on
a schedule, and closes invoices when a payment verifies. It never holds a key
and never signs.

**Tier 1: stock release binary, zero plugins, zero WASM.** Skills, SOPs, config,
and the built-in `http_request` tool. Nothing compiled, nothing installed beyond
the released artifact.

---

## Baixa is a ledger, not a till

A payment terminal's job ends the moment money arrives. Someone charges table 4,
the QR gets scanned, the payment lands, done. One state transition, one happy
path, and the terminal forgets you existed.

A freelancer's ledger runs for months and accumulates discrepancies nobody
notices. A client pays 180 against a 250 invoice and everyone assumes it settled.
A second transfer lands on the same reference and the duplicate is never
reconciled. Someone sends USDT to a USDC invoice. An invoice quietly ages past
sixty days because it never made it onto anyone's list. None of this is dramatic.
All of it is money.

Baixa is built for the second job. That is why it has four states instead of two,
six explicit failure paths, an aging report, and a human checkpoint on the one
outcome that deserves one.

Freelancers and small studios in Brazil invoicing in USDC live this loop: you
send a payment request, then you keep checking. Explorer tabs, wallet apps, a
spreadsheet that drifts. The work is not hard, it is constant, and it happens on
a phone between other tasks. Baixa moves it into Telegram, where the work already
is.

```
you → invoice Acme Studio 250 USDC for August

Baixa → 🧾 Invoice #7 — Acme Studio
        250 USDC · August
        Ref: 7f3aK2mQ…
        solana:<recipient>?amount=250&spl-token=<usdc>&reference=…
        [QR]

… two minutes after Acme pays …

Baixa → ✅ Invoice #7 — Acme Studio · 250 USDC paid
        https://solscan.io/tx/5xR2…
```

Every morning, one digest: paid today with a total, still open with age in days,
flagged items with reasons, running total received.

The interesting part is what happens when a payment is not clean.

---

## Four states, not two

| State | Meaning | Who sets it |
|---|---|---|
| `open` | Issued, nothing verified yet | `create_invoice`, at issue time |
| `paid` | Four on-chain conditions held | `reconcile`, from RPC only |
| `partial` | Right mint, right destination, short amount | `reconcile`, from RPC only |
| `flagged` | Something is wrong, with a recorded reason | `reconcile`, from RPC only |

`partial` is the state a till does not need and a ledger cannot do without. The
money arrived, at the right address, in the right token, and it is not enough.
That invoice is neither paid nor untouched, and treating it as either loses
real money.

`flagged` carries its reason verbatim into every daily digest until someone
deals with it: `wrong mint`, `destination mismatch`,
`duplicate payment: 2 matching transactions`.

---

## Custody: T1 — no signing keys

**Secrets held: Telegram bot token, LLM provider key, Solana RPC key. None can
move funds. The agent constructs payment URLs; the payer signs in their own
wallet.**

That is the precise version, and it is worth being precise: the custody ladder's
T1 line reads "Secrets held: none", which is true of *signing* material and not
true of the three credentials above. Baixa holds them, and none of them
authorizes a transfer.

There is no signing path anywhere in the build: no key material in config, none
in memory, and no tool on the agent's risk profile that could produce a
signature. `shell`, `file_write`, and `file_edit` are excluded on the risk
profile, so there is no route to a local key file or an external signer either.

No third-party MCP server, no facilitator, no custodial service. The only
external dependencies are a Solana RPC endpoint, an LLM provider, and
`api.qrserver.com` for QR rendering. `[http_request].allowed_domains` is narrowed
to exactly the RPC host and the QR renderer.

This is why the worst case of a successful prompt injection is one invoice
carrying a wrong address, rather than a drained wallet. THREAT_MODEL.md works
through that scenario without softening it.

---

## Correct layering: why this is T1 and not T2

Three questions decided the tier.

**Does the invoice need to be built by a plugin?** At T1 the `solana:` URL is
assembled by the model from operator constants, with a character-for-character
self-check against the literal before anything is written. A plugin would make
that structural instead of behavioral. It is a genuine improvement — and it is
also the only place in the design where a plugin clearly wins. THREAT_MODEL.md
§3 says so plainly rather than pretending the T1 check is airtight.

**Does verification need a plugin?** No, and this is the load-bearing answer.
Verification is four boolean checks against a `getTransaction` response:
`meta.err` is null, an SPL transfer exists with the right mint, the destination
*owner* matches the recipient, and the amount clears the invoice. Four
comparisons. Wrapping that in WASM would add a build step, a distribution
problem, and a version-skew surface, and would buy nothing. `http_request` plus a
SOP does it on the stock binary.

**Does scheduling need a plugin?** No. ZeroClaw's SOP engine already runs
cron-triggered, multi-step, auditable procedures with fail-closed error handling
and durable run state. Reimplementing that in a plugin would be strictly worse.

So: a T1 problem got a T1 solution. The one place a plugin would genuinely help
is named, scoped, and left for a follow-up rather than bolted on to look
sophisticated.

**Evidence the stock artifact suffices.** Release binaries build against the
`dist` feature selection
(`.github/workflows/release-stable-manual.yml:319,328`), which resolves to the
expansion of `default` plus `dist_extra_features`
(`xtask/src/generate/spec.rs:673-686`). `default` includes `agent-runtime` and
`default-channels` (`Cargo.toml:282-289`), and `default-channels` includes
`channel-telegram` (`Cargo.toml:306-310`). `dist_extra_features` is
`channel-matrix`, `channel-lark`, `whatsapp-web` (`Cargo.toml:259-263`).
`plugins-wasm` appears in neither (`Cargo.toml:394`, `:427-429`). The released
binary carries no WASM plugin host, and Baixa needs none.

---

## How it is built

| Piece | What it is | File |
|---|---|---|
| `create_invoice` | Skill. Parses free-form input, mints a reference, builds the URL, appends to the ledger, replies with a QR. | `skills/create_invoice/SKILL.md` |
| `reconcile` | SOP walked by a `[cron.reconcile]` agent job every two minutes. Polls RPC, verifies four conditions, updates status, notifies, parks on a human checkpoint for destination mismatch. | `sops/reconcile/` |
| `daily_digest` | SOP, cron `0 8 * * *` (09:00 at UTC+1). Read-only summary. | `sops/daily_digest/` |
| Config | Provider, agent, risk profile, Telegram, peer group, bundle, SOP dir, HTTP allowlist, memory. | `config.example.toml` |

Every config key is cited to `crates/zeroclaw-config/src/schema.rs`, and every
SOP manifest field to `crates/zeroclaw-runtime/src/sop/types.rs`, because most of
the operator-facing reference does not exist as prose in the repository — see
SETUP.md §9.4.

### The ledger is one record

Every invoice lives in a single memory entry, key `baixa_ledger`, category
`core`, holding a JSON array. Each write rewrites the whole array; reconcile does
one recall and iterates in memory.

This is deliberate. `memory_recall` is a scored relevance search, not a query
(`crates/zeroclaw-tools/src/memory_recall.rs:21-56`), and under `hybrid` mode a
`min_relevance_score` of 0.4 silently drops low scorers
(`schema.rs:10621-10625`). Per-invoice records would mean "list all open
invoices" depends on relevance ranking — which is exactly the kind of quiet,
occasional data loss you do not want in an accounts-receivable ledger. One
record sidesteps it entirely.

### The failure-path matrix

Every one of these has an explicit rule in `sops/reconcile/SOP.md`. None of them
is a catch-all.

| Condition | Status effect | Operator sees |
|---|---|---|
| RPC timeout | Unchanged, retry in 2 min | log line |
| RPC error object | Unchanged, retry | log line |
| Malformed / truncated JSON | Unchanged, retry | log line |
| `getTransaction` returns `null` | Unchanged (not yet available) | log line |
| `confirmationStatus: processed` | Unchanged (unconfirmed) | log line |
| `meta.err` non-null | Unchanged (tx failed on-chain) | log line |
| No signatures on the reference | Unchanged (nobody paid yet) | nothing |
| Amount short | → `partial`, stays open | `⚠` with shortfall |
| Amount over | → `paid`, overage noted | `✅` + overage |
| Wrong mint | → `flagged` `wrong mint` | `🚩` |
| Wrong destination owner | → `flagged` `destination mismatch` | `🚩` + **human checkpoint** |
| Two verifying signatures | → `paid` on earliest + duplicate flag | `✅` + `🚩` |

Every path either derives a status from four verified RPC conditions or leaves
the status exactly as it was and retries in two minutes. Nothing is ever marked
paid on incomplete evidence.

One detail worth calling out: condition 3 compares the destination **owner**,
read from `meta.postTokenBalances[].owner`, not the destination token-account
address. A token account is not a wallet, and comparing the wrong one is the
kind of bug that passes a demo and fails in production.

### One human checkpoint, on the one outcome that earns it

`destination mismatch` means an invoice went out carrying an address that is not
the operator's. Everything else in Baixa runs unattended; this parks.

Step 7 of `sops/reconcile/SOP.md` is a `kind: checkpoint` step with
`requires_confirmation: true`. Step 6 carries the routing guard
`when: $.steps.5.destination_mismatch == "true"`, so on a clean run the
checkpoint is never dispatched and nobody is paged.

The gate is not decorative. `[sop] approval_mode = "out_of_band_required"` turns
the agent's own `sop_approve` into a no-op, leaving only a CLI / WS / HTTP
principal able to clear it (`schema.rs:22718-22720`) — under the default `both`,
the agent could have approved itself (`schema.rs:22714-22717`). A gate nobody
answers re-surfaces forever and never self-approves
(`approval_timeout_action = "escalate"`, `schema.rs:22733-22736`).

The ledger write in step 5 happens *before* the gate, so the flag is durable
whether or not anyone answers. Approving acknowledges; it does not clear the
flag or change a status.

---

## Things that cost time, written down

Every item below was found by **running the thing**, not by reading about it.
That distinction is the point: each one passed `zeroclaw doctor`, `sop validate`,
and `skills list` while being broken, and several produced no error at any log
level. Reading the docs and the schema is what got the config written; running it
is what found out the config was wrong.

Full detail in SETUP.md §9 and in the commit messages, each of which cites the
source line that explains the behaviour.

### A SOP cron trigger fires and then nothing executes it

This is the one that would have sunk the project silently.

The docs contradict each other on whether SOP cron works at all:
`docs/book/src/sop/fan-in/cron.md:3` says triggers are **wired**;
`docs/book/src/reference/feature-matrix.md:64-67` says they are "defined and
matched but not yet routed to a live source." Reading the source settles the
first question — `src/main.rs:7829` spawns the maintenance task,
`sop/dispatch.rs:1233` builds the schedule cache, `dispatch.rs:1311-1343`
dispatches on schedule. Cron fires. `feature-matrix.md` is stale.

**Firing is not executing.** The daemon hands the dispatch result to
`process_headless_results` (`src/main.rs:7952`), which is a logger: every branch
records a line and returns. For an LLM step it records

```
run ... ready for step 1 'Load open invoices' but no agent loop available to execute
```

at INFO, and leaves the run `Running` forever. `record_started_run`
(`sop/dispatch.rs:283-289`) states it outright: only
`execution_mode = "deterministic"` is driven to a terminal state, because
deterministic steps need no model. Everything else holds its `max_concurrent`
slot indefinitely, so the SOP fires exactly once and every later tick is skipped
with `cooldown or concurrency limit reached`.

Confirmed against the run store rather than the log: `runs.db` held the run with
`terminal=0` and a live row in `sop_claims`, lease an hour out. `sop deny` returns
`already_resolved` on such a run because it is Running, not awaiting approval.

Deterministic mode is not open to this SOP either. The builtin capabilities are
`approval.wait`, `json.validate`, `shell.exec`, `git.status`, `git.diff` and
`notify.channel` (`sop/capability/builtins.rs:102-244`). None speak HTTP, and
reconcile exists to query Solana RPC.

Baixa therefore schedules through a declarative `[cron.<alias>]` agent job
(`CronJobDecl`, `schema.rs:12687-12723`), which runs inside a real agent turn and
walks the SOP with `sop_execute` / `sop_advance`. The SOPs keep their steps,
their output contracts, and the step 7 checkpoint. Two things improve on the way
through: `allowed_tools` on a cron job is a hard allowlist handed to the agent
run (`cron/scheduler.rs:822`), so `daily_digest` having no `http_request` and no
`memory_store` stops being an instruction and becomes a runtime constraint; and
`[cron]` schedules carry `tz`, which the SOP trigger does not, so the digest
reads `0 9 * * *` in a named zone instead of a UTC hour with a comment.

### A skill can be installed, listed, and still never reach the agent

`create_invoice` never once entered the system prompt. The agent eventually said
so itself over Telegram: *"I don't have an SOP or skill configured yet to create
invoices."*

`[skill_bundles.<alias>] include` is compared against the skill's `name:`
frontmatter field, not its directory name (`skills/mod.rs:652` —
`if !bundle.admits_skill(&skill.name) { continue; }`). The directory was
`create_invoice`, the frontmatter said `create-invoice`, the include list said
`create_invoice`. One character. The skill was filtered out of every turn,
silently, with no warning at any log level.

What made it expensive: **`zeroclaw skills list` does not apply the include
filter.** It walks the bundle directory and reports what it finds, so it showed
the skill as installed for the entire session. A green `skills list` is not
evidence that an agent has the skill.

The cost of not knowing this was three wrong diagnoses in a row. The model was
blamed twice for improvising, and the identity prompt once for framing the agent
as a reconciler. The agent was improvising because it genuinely had no
instructions; with none to follow, its identity was the only thing left to act
on. The tell, in hindsight, is unmistakable: **an agent that denies having a
capability you configured has a name mismatch, not a prompt problem.**

### A classifier decides whether your message deserves a reply

An explicit `invoice Acme Studio 250 USDC for August` over Telegram produced
silence. No error, nothing at WARN or above. The trace shows the message
arriving and then:

```
reply-intent precheck completed
channel_message_no_reply
```

Before the agent loop runs, a separate classifier model call decides whether an
inbound message warrants a response at all
(`channels/orchestrator/mod.rs:4950-4994`, `ChannelPrecheckConfig` at
`scattered_types.rs:341-354`). It defaults to enabled, and it classified an
invoice request as not needing a reply. The agent loop never ran, so nothing
downstream had a chance to fail loudly.

Off, via `[agents.baixa.precheck] enabled = false`. Baixa has exactly one
authorized sender, pre-authorized by numeric ID. Every message reaching it is by
definition for it, and a classifier that can decide otherwise is a silent failure
mode on the issuing path.

### A three-name denylist is not a tool policy

The risk profile originally carried `excluded_tools = [shell, file_write,
file_edit]`, and THREAT_MODEL.md claimed on that basis that the agent had no
filesystem access. A live run showed it calling `file_read` and `glob_search` and
offering to browse the workspace for an invoices directory.

Two defects in one line. A three-name denylist admits every tool nobody thought
to name. And `excluded_tools` is scoped: the schema documents it as "Tools
excluded from **non-CLI channels** under this profile"
(`schema.rs:11841-11847`), so it never applied to `zeroclaw agent` at all.

Now `allowed_tools` names the seven tools Baixa needs
(`schema.rs:11812-11840`); anything unnamed is unreachable. Startup confirms it:
55 registered tools narrowed to 7 for the agent, and to 5 inside the
`daily_digest` cron job. `excluded_tools` stays as a redundant second statement
about the three worst offenders, not as the mechanism.

This one is also a correction to an earlier version of this repository's own
threat model, which is why it is written down rather than quietly fixed.

### `uses_memory = false` deletes the memory tools

The name reads like "do not inject recalled context." The scheduler maps it to
`memory_free` (`cron/scheduler.rs:795-800`), which installs a null Memory
(`agent/loop_.rs:1230`) and sets `exclude_memory` (`:1333`), which strips every
`MEMORY_TOOL_NAMES` entry from the registry (`tools/scoped.rs:238`).

The Baixa ledger is a memory record, so this deleted the ledger. The visible
symptom was three layers away: `memory_recall` returned `Unknown tool`, the agent
reported the step failed with a prose reason, and the engine then rejected that
prose against the step's output contract. What surfaced was a schema mismatch.
Two rounds of prompt edits went into the wrong layer before the tool trace showed
the real failure.

### A transport error that reads as "nobody paid"

Live reconcile runs were getting `HTTP 415` on every RPC call. ZeroClaw's
`http_request` does not set a Content-Type of its own, and the tool exposes
`headers` for exactly this (`zeroclaw-tools/src/http_request.rs:460-464`).
Confirmed directly against the endpoint: no header returns 415, `Content-Type:
application/json` returns 200.

The worse half was the agent's reaction. Step 2 answered a 415 with
`{"candidates": []}` — the same output it produces when nobody has paid. A
transport failure and a genuinely unpaid invoice became indistinguishable, so a
broken or rate-limited endpoint would have held every invoice open indefinitely
while the run reported success and the digest reported nothing wrong. The
file-level fail-closed rule was not enough because it lived several screens above
the step; both request steps now say at the point of action that a non-2xx status
fails the step.

### The ledger disappears from its own memory store

The worst of the eleven, because it fails while reporting success.

The Baixa ledger is one memory record under the key `baixa_ledger`. Step 1 of
reconcile reads it with `memory_recall`. After a few hours of running, that
recall stopped returning the ledger and started returning **this SOP's own past
runs**, and reconcile began reporting `{"pending": []}` while two invoices sat
open.

The mechanism is a feedback loop. `SopAuditLogger` writes one record per step
into the same memory store as the ledger, unconditionally — there is no config
switch and the daemon always constructs it against live memory
(`src/main.rs:9601`). Each record embeds that step's tool calls *together with
their outputs*. So the output of a `memory_recall` for `baixa_ledger` becomes
searchable text containing the string `baixa_ledger` several times over, while
the real ledger record contains it only in its key and holds invoice JSON in its
body. `memory_recall` is BM25 with no exact-key or category filter
(`zeroclaw-tools/src/memory_recall.rs:29-56`). Audit records therefore outrank
the ledger for its own key, and each cycle writes a denser one.

What saved the data was a rule already in step 1: treat any result whose key is
not exactly `baixa_ledger` as no ledger at all. It fired correctly and returned
an empty pending list. Nothing was corrupted. Nothing was reconciled either, and
every run reported `completed`.

Mitigated at Tier 1 by raising the recall limit to 25 and selecting the entry by
exact key, with an explicit instruction that a stale `{"pending": []}` recovered
from a past step result is indistinguishable from a healthy empty ledger. That is
a mitigation, not a fix: at 720 runs a day the audit will eventually push the
ledger past rank 25 as well. The honest statement is that a ledger does not
belong in a shared searchable store, and Tier 1 offers no other one.

### A cost ceiling that only covers models it has prices for

`[cost] daily_limit_usd = 3.00` with `enforcement.mode = "block"` was added
specifically so an agent firing 720 times a day could not run up an unbounded
bill. It did not work.

The block fires when the runtime believes the limit is reached, and the runtime
prices a call from `[cost.rates.*]`. A model absent from that sheet costs
$0.00 — so the limit is never reached.

The sheet listed `claude-opus-5` and `claude-haiku-4-5`. The provider model was
switched to `claude-sonnet-4-5` and no row was added. From
`state/costs.jsonl`:

| Model | Calls | Input tokens | Tracked | Real |
|---|---:|---:|---:|---:|
| `claude-sonnet-4-5` | 233 | 3,016,851 | $0.00 | **$9.28** |
| `claude-haiku-4-5` | 195 | 1,318,153 | $0.64 | $1.37 |
| `claude-opus-5` | 26 | 128,966 | $0.40 | $0.80 |
| | | | **$1.04** | **$11.44** |

The counter read a third of the ceiling while the real bill was nearly four times
it. Nothing surfaced it: a $0.00 model is indistinguishable from a cheap one in
`zeroclaw status`.

This one is ours, not the platform's. The ceiling was presented in an earlier
version of this repository as a real control, and switching models without
updating the rate sheet made it inert for the only model in use.

The same data locates the actual spend: 3.0M input tokens across 233 calls is
roughly 13,000 per call, which is the system prompt with every skill body inlined
at `prompt_injection_mode = "full"`, re-sent every time. The dominant cost of a
scheduled agent is re-transmitting its instructions.

### Bundle skills never appear in the Telegram command menu

The bot had no product surface. Pressing `/` in Telegram listed 13 built-in
runtime commands and nothing about invoicing, while three skills were installed
and working.

`register_bot_commands` builds that menu from `load_skills(workspace_dir)`
(`channels/telegram.rs:1010`) and registers each skill's `name` and `description`
as a `/command` via `setMyCommands` (`:988-1069`). It reads the **workspace**
skills directory only. Bundle skills reach the agent's system prompt and never
reach the menu, so a bundle-only install is fully functional and completely
invisible to the operator.

Both sources load into the prompt and workspace wins a name collision
(`skills/mod.rs:655-660`), which makes "install in both" a silent duplication
with a drift hazard — the same shape as the `include` mismatch above. Baixa keeps
skills in the workspace directory and has no bundle.

### No free provider can run this agent, and the reason is arithmetic

Worth knowing before anyone plans a ZeroClaw project around a free tier.

Groq is the obvious candidate: free without a card, OpenAI-compatible, supported
by ZeroClaw at `[providers.models.groq.<alias>]`. It was tried properly and it
does not work, for three independent reasons found in this order.

`openai/gpt-oss-120b` breaks the tool protocol: Groq returns
`400 tool_choice is none, but model called a tool`, and the attempted call is
named `tool.sop_status` rather than `sop_status`.

`llama-3.3-70b-versatile` emits the wrong syntax entirely — llama's *text*
function format inside a native tool-calling request,
`<function=sop_execute={...}</function>`, rejected as `tool_use_failed`. Setting
`native_tools = false`, which is ZeroClaw's default for Groq precisely because of
this family's history, does not help.

The third one ends the discussion. **The free tier caps at 12,000 tokens per
minute; this agent's system prompt is roughly 13,000 tokens per call.** Measured,
not estimated: 3.0M input tokens across 233 calls in `state/costs.jsonl`, every
skill body inlined at `prompt_injection_mode = "full"`. A single turn does not fit
inside the per-minute quota. That is not a burst to back off from — the request is
larger than the budget it draws against.

The general form, which outlives Groq: **any free tier capped below roughly 15k
tokens per minute cannot run a Tier 1 ZeroClaw agent that inlines its skills.**
The lever is `prompt_injection_mode = "compact"`, which keeps skill bodies out of
the prompt and fetches them through `read_skill` on demand. It would fit. It also
adds a mandatory tool call to every turn, on models that had just demonstrated
they mangle ordinary ones.

### Approval matches on tool name and ignores the HTTP method

`docs/book/src/tools/overview.md:89-93` describes a low/medium/high risk model
where `http_request` GET is Low and POST is High, and says supervised autonomy
runs low, asks on medium, blocks high.

The implementation never inspects the method.
`ApprovalManager::approval_requirement`
(`crates/zeroclaw-runtime/src/approval/mod.rs:175-211`) checks `Full` → approved,
`ReadOnly` → not required, `always_ask` → prompt, `auto_approve` → approved,
session allowlist → approved, else prompt (`approval/mod.rs:209-210`).

Solana JSON-RPC is POST. Under the default `supervised`, every RPC call would
prompt an operator who is not there at 03:00. Baixa keeps
`level = "supervised"` and names the tools explicitly in `auto_approve` rather
than escalating to `full`, so anything unlisted still gates.

### A peer group that authorizes inbound does not authorize outbound

`reconcile` notifies the operator with `send_message_to_peer`, which resolves
its `target` through `resolve_peer_set`. That function skips any peer group
which does not list the calling agent
(`crates/zeroclaw-runtime/src/peers.rs:62-69`).

The group `zeroclaw channel bind-telegram` creates automatically has no `agents`
field (`docs/book/src/channels/telegram.md:216`), so it authorizes inbound
Telegram messages and silently provides no valid outbound target. The symptom is
an agent that answers you fine and never sends an unprompted notification.

Baixa configures one group carrying both `agents` and `external_peers`, which
covers inbound authorization and outbound addressing at once.

### There is no supported home for a custom config section

The recipient address had to come from config. The root `Config` struct
(`schema.rs:76-78`) has no serde catch-all and no `deny_unknown_fields`, so a
`[baixa]` section loads silently, is dropped from the typed struct, and is erased
by the next `Config::save()` (`schema.rs:21370-21374`) — which
`zeroclaw channel bind-telegram` calls (`orchestrator/mod.rs:7131`), as does the
in-Telegram `/bind` flow. The address would disappear the first time the operator
authorized a user.

The typed field that does survive is `[agents.<alias>.identity].aieos_inline`
(`IdentityConfig`, `schema.rs:6407-6417`), parsed at `identity.rs:189` and
rendered into the system prompt by `aieos_to_system_prompt` (`identity.rs:719`,
with `bio` emitted verbatim at `:742-743`). Baixa puts the constants there, and
independently in the skill file and the SOP files: three operator-written copies,
none of them reachable from chat.

---

## What is deliberately not claimed

- **Invoice creation is model-mediated.** A strong enough injection could produce
  a wrong address at issue time. Blast radius is one invoice, caught by reconcile
  within two minutes as `destination mismatch`, because Baixa holds no keys and
  the payer signs. THREAT_MODEL.md §3.
- **The payment reference is model-generated, not from a CSPRNG.** No shell, no
  crypto tool at T1. Collisions are checked against the ledger before use.
  Consequence is noise, not theft. THREAT_MODEL.md §4.
- **Raw `getTransaction` JSON enters the model context for exactly one step.**
  `http_request` returns its body to the calling turn by construction
  (`crates/zeroclaw-tools/src/http_request.rs:478-489`) and there is no
  interception point without a plugin. Baixa keeps it inside one step, out of
  memory, and away from the operator. Genuinely shaping it to ~200 tokens before
  it hits context is a T2 job. THREAT_MODEL.md §6.
- **Step output contracts are advisory, not enforced.** `[sop]
  step_schema_enforce` defaults to `true` and fails any step whose final message
  is not exactly the declared shape (`schema.rs:22835-22837`). On the
  agent-driven path that message is whatever the model last said, and observed
  rejections included a correct object wrapped in a ```` ```json ```` fence and a
  correct object preceded by one sentence of correct reasoning. Both got the work
  right and the wrapper wrong. Failing a payments reconciliation over a stray
  sentence is worse than proceeding, so enforcement is off and the contracts
  remain as documentation plus best-effort structure. Parsing into run data does
  not depend on the flag (`sop/rundata.rs:67-85`), so the step 5 routing value
  still populates; fail-closed behaviour comes from the step instructions and
  per-step `on_failure: fail`, neither of which depends on formatting.

Each of those is a real gap with a named fix. None of them touches custody.

---

## Repository

```
config.example.toml          Annotated config, every key cited to the schema
.gitignore                   Excludes the real config, logs, secrets, DBs
skills/create_invoice/       SKILL.md — the issuing skill
sops/reconcile/              SOP.toml + SOP.md — 2-minute verification loop
sops/daily_digest/           SOP.toml + SOP.md — daily summary
SPEC.md                      Stage 1 platform recon, file:line throughout
THREAT_MODEL.md              What holds, what does not, and why
INJECTION_TEST.md            Four attacks, each with its structural reason
SETUP.md                     Install, run, and the docs-vs-source log
SUBMISSION.md                This file
```

Start with SETUP.md to run it, THREAT_MODEL.md to judge it.
