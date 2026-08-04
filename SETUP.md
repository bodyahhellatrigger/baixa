# SETUP.md

Getting Baixa running on a stock ZeroClaw release binary. No plugins, no WASM,
no source build.

Citations point at the ZeroClaw source tree. Clone it alongside this repo if you
want to follow them:
`git clone --depth 1 https://github.com/zeroclaw-labs/zeroclaw.git`

---

## 1. Install ZeroClaw

```sh
curl -fsSL https://raw.githubusercontent.com/zeroclaw-labs/zeroclaw/master/install.sh | bash
```

(`README.md:36-38`)

The prebuilt release binary already carries everything Baixa needs. Release
artifacts build against the `dist` feature selection
(`.github/workflows/release-stable-manual.yml:319,328`), which resolves to the
expansion of `default` plus `dist_extra_features`
(`xtask/src/generate/spec.rs:673-686`). That includes `agent-runtime` and
`channel-telegram` (`Cargo.toml:282-289`, `:306-310`) and excludes
`plugins-wasm` (`Cargo.toml:394`, `:427-429`). Nothing here needs a source
build.

### If `zeroclaw` is not found afterwards (macOS + bash)

The installer puts the binary in `~/.cargo/bin` and appends the `PATH` export
to `~/.bashrc`. Terminal.app starts bash as a **login** shell, and a login
bash reads `~/.bash_profile` and never `~/.bashrc`. If your `.bash_profile`
does not source `.bashrc`, the export never runs and every `zeroclaw` command
reports `command not found` even though the binary is installed.

```sh
cat >> ~/.bash_profile <<'EOF'
if [ -f "$HOME/.bashrc" ]; then
  . "$HOME/.bashrc"
fi
EOF
```

Open a new terminal, then check:

```sh
zeroclaw --version    # zeroclaw 0.8.3
```

## 2. Create the Telegram bot

Open [@BotFather](https://t.me/BotFather), send `/newbot`, follow the prompts,
copy the token (`docs/book/src/channels/telegram.md:33-39`).

Treat the token like a password. Do not paste it into `config.toml`, logs, or
screenshots (`telegram.md:41-42`).

## 3. Lay down the files

```sh
INSTALL=~/.zeroclaw

# Skill bundle. With `directory` omitted in [skill_bundles.baixa], ZeroClaw
# resolves the bundle to <install>/shared/skills/baixa/ (docs tools/skills.md:11).
mkdir -p "$INSTALL/shared/skills/baixa"
cp -r skills/create_invoice "$INSTALL/shared/skills/baixa/"

# SOPs. Each SOP is a directory with SOP.toml plus optional SOP.md
# (docs sop/syntax.md:5-14).
mkdir -p "$INSTALL/workspace/sops"
cp -r sops/reconcile sops/daily_digest "$INSTALL/workspace/sops/"

# Config
cp config.example.toml "$INSTALL/config.toml"
```

## 4. Fill in the placeholders

Edit `~/.zeroclaw/config.toml` and replace every `<PLACEHOLDER>`:

| Placeholder | Value |
|---|---|
| `<ANTHROPIC_API_KEY>` | your provider key |
| `<RECIPIENT_SOLANA_ADDRESS>` | the wallet that receives USDC |
| `<USDC_MINT_ADDRESS>` | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` on mainnet |
| `<HELIUS_API_KEY>` | your Helius API key, inside `SOLANA_RPC_URL` in `aieos_inline` |
| `<OPERATOR_TELEGRAM_NUMERIC_ID>` | your numeric Telegram user ID |
| `<ABSOLUTE_PATH_TO>/sops` | absolute path, e.g. `/home/you/.zeroclaw/workspace/sops` |
| `<ABSOLUTE_PATH_TO_WORKSPACE>/brain.db` | absolute path, e.g. `/home/you/.zeroclaw/data/brain.db` |

**Then do the same in three more places.** The recipient and mint appear in
three independent operator-written files and all three must agree:

1. `~/.zeroclaw/config.toml` → `[agents.baixa.identity].aieos_inline`
2. `~/.zeroclaw/shared/skills/baixa/create_invoice/SKILL.md` → OPERATOR CONSTANTS
3. `~/.zeroclaw/workspace/sops/reconcile/SOP.md` → OPERATOR CONSTANTS
   (and `daily_digest/SOP.md`, which keeps a copy for cross-checking)

THREAT_MODEL.md §3 explains why the address lives in three files rather than one
custom config section.

Set the bot token through the masked prompt rather than by editing the file
(`telegram.md:52-55`):

```sh
zeroclaw config set channels.telegram.home.bot_token
```

`bot_token` is a `#[secret]` field and is stored encrypted
(`crates/zeroclaw-config/src/schema.rs:13749-13760`).

### Do not delete `[storage.sqlite.default]`

`[memory] backend = "sqlite"` does not, on its own, give the agent a memory.
`resolve_active_storage` (`schema.rs:4446-4458`) splits that string into a kind
and an alias, defaults the alias to `default`, and looks the pair up in
`[storage.sqlite]`. When the alias is absent it returns `ActiveStorage::None`
and **does not raise an error**.

The whole Baixa ledger lives in memory, so this failure removes the ledger.
Nothing in `zeroclaw doctor`, `config list`, or `sop validate` reports it. The
only signal is one word in the daemon banner:

```
  🧠 Memory:   none (auto-save: off)      <- broken, ledger has nowhere to live
  🧠 Memory:   sqlite (auto-save: off)    <- correct
```

Read that line every time you start the daemon.

### Why both SOPs set `execution_mode = "auto"`

`[sop] default_execution_mode` is left at `supervised`, and each SOP overrides
it in its own `SOP.toml`. That looks backwards until you read what supervised
does: `execution_mode_needs_approval` (`sop/engine.rs:5456-5459`) returns true
for `step.number == 1`, so supervised means **an approval prompt before step 1
of every single run**, not "run, but carefully".

A cron SOP has nobody to answer that prompt. On the first live start every
reconcile tick parked at step 1 and the next tick was refused with
`cooldown or concurrency limit reached`, because `max_concurrent = 1` counted
the parked run. `approval_mode = "out_of_band_required"` compounds it: the
agent cannot clear its own gate, so the runs accumulate until a human clears
each one from the CLI.

Switching to `auto` does not give up the gate this project actually needs.
`step_requires_approval_gate` (`sop/engine.rs:5473-5476`) returns true for any
step carrying `requires_confirmation` **before** it consults the mode, and
`pending_step_blocks_direct_advance` (`:5483-5485`) does the same for
`kind: checkpoint`. Step 7 of reconcile carries both, so the
destination-mismatch checkpoint still stops the run under `auto` while steps 1
to 6 run unattended.

Leave the global default at `supervised` so any SOP later dropped into the
directory is gated until someone opts it out deliberately.

If a run is stuck holding the concurrency slot:

```sh
zeroclaw sop pending
zeroclaw sop deny <run-id>
```

### The bundle `include` list matches the skill NAME, not the directory

`[skill_bundles.baixa] include` is compared against each skill's `name:`
frontmatter field, not against its directory name
(`skills/mod.rs:652` — `if !bundle.admits_skill(&skill.name) { continue; }`).

This cost most of a debugging session. The directory was `create_invoice`, the
frontmatter said `name: create-invoice`, and the include list said
`create_invoice`. The names never matched, so the skill was dropped from the
agent's prompt on every turn — silently, with no warning at any log level.

What made it hard to see: **`zeroclaw skills list` does not apply the include
filter.** It walks the bundle directory and reports what it finds, so it showed
`create-invoice v0.1.0` as installed the whole time the agent could not see it.
A green `skills list` is not evidence that an agent has the skill.

The symptom at the other end was an agent that improvised: asked the operator
for a transaction signature to "reconcile" an issuing request, offered to draft
a workflow, and eventually said outright that it had no skill configured for
creating invoices. That last reply is the reliable tell — if the agent denies
having a capability you configured, check the name match before touching
prompts or models.

Keep the directory name, the `name:` frontmatter, and the `include` entry
identical. All three are `create_invoice` here.

## 5. Authorize yourself

`[peer_groups.baixa_operator]` in the example config pre-authorizes you. Put
your numeric Telegram user ID in `external_peers`. A numeric ID is preferable to
a username because it survives a rename (`telegram.md:104-106`).

If you do not know your ID, leave `external_peers` empty and use the pairing
flow instead: start the daemon, watch for
`Telegram pairing required; one-time bind code issued` in the output, then send
`/bind <code>` to the bot from the account you want approved
(`telegram.md:98-100`, `:157-163`).

**Never use `external_peers = ["*"]`.** It accepts every sender who can reach
the bot and disables pairing (`telegram.md:121-126`).

Note the group needs both `agents = ["baixa"]` and `external_peers`.
`resolve_peer_set` skips any group that does not list the calling agent
(`crates/zeroclaw-runtime/src/peers.rs:62-69`), so the group that
`bind-telegram` auto-creates — which has no `agents` — authorizes inbound
messages but does not make you a valid `target` for `send_message_to_peer`. The
reconcile notifications need the latter.

## 6. Validate before starting

```sh
zeroclaw sop validate
zeroclaw sop list
zeroclaw skills list --agent baixa
zeroclaw channel doctor
```

`sop validate` warns on empty names, missing triggers, missing steps, and step
numbering gaps (`docs/book/src/sop/syntax.md:434-441`).

`skills list --agent baixa` shows what the agent actually loads at runtime, as
opposed to what is merely installed (`docs/book/src/tools/skills.md:106-116`). If
`create_invoice` does not appear here, the bundle is not attached — check
`agents.baixa.skill_bundles`.

## 7. Start

```sh
zeroclaw daemon
```

The daemon is required, not optional. See §8.

For a long-running install:

```sh
zeroclaw service install
zeroclaw service start
zeroclaw service logs --follow
```

---

## 8. Operational rules you cannot skip

### The daemon must be running

SOP cron triggers fire from the SOP maintenance tick, which only
`zeroclaw daemon` or the `zeroclaw channel start` supervisor spawns. A
standalone `zeroclaw gateway start` does **not** spawn it and will not fire
anything (`docs/book/src/sop/fan-in/cron.md:3`).

### Restart the daemon after editing a SOP

**Cron schedules are parsed once, at daemon start.** `SopCronCache::from_engine`
builds the schedule cache when the maintenance task spawns
(`src/main.rs:7839-7841`, cache built at
`crates/zeroclaw-runtime/src/sop/dispatch.rs:1233`). A SOP added or edited while
the daemon is running does not take effect until a reload
(`sop/fan-in/cron.md:3`).

So after any change under `sops/`:

```sh
zeroclaw service restart     # or stop and restart `zeroclaw daemon`
```

This bites silently. The SOP validates fine, `sop list` shows it, and it never
fires.

### Keep `maintenance_interval_secs` below your cron period

The tick is a poller, not a per-schedule timer. Its period is
`[sop] maintenance_interval_secs`, default 60
(`crates/zeroclaw-config/src/schema.rs:22586-22593`). Setting it to `0` disables
the tick entirely.

Baixa's `*/2 * * * *` needs the tick strictly below 120 seconds. The default 60
is fine. A sub-minute cron would not be.

Firing is also at-most-once per window: if several occurrences fall inside one
poll interval, the SOP fires once, by design
(`dispatch.rs:1325-1333`). At two minutes this never matters.

### Config edits need a reload too

A direct `config.toml` edit or a standalone `zeroclaw config set` takes effect
only after a daemon reload or process restart. Saving alone does not rebuild
long-running listeners (`telegram.md:221-228`).

### One process per bot token

Two processes polling the same token produce `Telegram polling conflict (409)`
(`telegram.md:257`).

### Clearing the destination-mismatch checkpoint

`reconcile` parks on a human checkpoint when it flags a `destination mismatch`
(step 7 of `sops/reconcile/SOP.md`). Nothing else in Baixa waits for a person.

```sh
zeroclaw sop pending          # list parked runs
zeroclaw sop approve <run_id>
```

`[sop] approval_mode = "out_of_band_required"` in the example config makes the
agent's own `sop_approve` a no-op (`schema.rs:22718-22720`), so the gate has to
be cleared from the CLI or another out-of-band surface. Under the default
`both`, the agent could clear it itself (`schema.rs:22714-22717`) and the
checkpoint would be decorative.

A gate nobody answers never self-approves: `approval_timeout_action = "escalate"`
re-surfaces it and keeps waiting (`schema.rs:22733-22736`).

Approving acknowledges the flag. It does not clear the flag, change a status, or
make the invoice payable.

### Digest timing is UTC, with no DST

`daily_digest` fires `0 8 * * *`, which is 09:00 for an operator at UTC+1.

The SOP cron trigger variant has no `tz` field
(`crates/zeroclaw-runtime/src/sop/types.rs:161-164`) — unlike declarative
`[cron.<alias>]` jobs, which do (`schema.rs:12776-12780`). Window evaluation runs
against `chrono::Utc::now()` (`dispatch.rs:1317`), so the offset is baked into
the hour by hand.

Two consequences: operators in other zones must edit the hour, and the digest
does not follow daylight saving. At UTC+2 in summer it arrives at 10:00 local
until you change it.

---

## 9. Docs vs source

Four places where the ZeroClaw documentation disagrees with the code, or is
absent. Each cost real time; recording them here so the next person skips that.

### 9.1 SOP cron triggers: "wired" vs "not wired"

- `docs/book/src/sop/fan-in/cron.md:3` — **Wired.** "Cron triggers are dispatched
  by the periodic SOP maintenance tick."
- `docs/book/src/reference/feature-matrix.md:64-67` — cron triggers are "defined
  and matched but not yet routed to a live source", making the capability
  experimental.

These cannot both be true, and Baixa's entire reconcile loop depends on the
answer.

**Resolved by reading the source.** `src/main.rs:7829` defines
`spawn_sop_maintenance`, which builds a `SopCronCache` (`main.rs:7839-7841`) and
loops on a ticker calling `run_sop_maintenance_tick` (`main.rs:7842-7847`,
`:7899`). `SopCronCache::from_engine`
(`crates/zeroclaw-runtime/src/sop/dispatch.rs:1233`) parses each SOP's cron
expression, and `check_sop_cron_triggers` (`dispatch.rs:1311-1343`) dispatches a
`SopEvent` for every expression whose occurrence fell in the window
`(last_check, now]`.

**`cron.md` is correct. `feature-matrix.md:64-67` is stale.**

Also confirmed at `dispatch.rs:1252-1253`: a 5-field crontab is normalized to
6-field by prepending a seconds field, so `*/2 * * * *` parses without a leading
seconds column.

### 9.2 `SKILL.toml` is deprecated, the docs still recommend it

`docs/book/src/tools/skills.md:17` presents `SKILL.md` and `SKILL.toml` as
co-equal recommended local authoring formats.

The code disagrees. `crates/zeroclaw-runtime/src/skills/constants.rs:4` sets the
canonical manifest filename to `SKILL.md`; line `:8` lists `SKILL.toml` and
`manifest.toml` as deprecated — "still accepted by the audit loader for
back-compat with installed skills. Never written by the service."

Baixa uses `SKILL.md`.

The frontmatter field list also differs: `skills.md:65` names `name`,
`description`, `version`, `author`, `tags`. The struct
(`crates/zeroclaw-runtime/src/skills/frontmatter.rs:12-27`) also accepts
`license`, `category`, and `slash_options`.

### 9.3 Approval keys on tool name, not HTTP method

`docs/book/src/tools/overview.md:89-93` describes a low/medium/high risk model in
which "`http_request` GET to allowed domains" is Low, "`http_request` POST to
unconstrained URLs" is High, and "Default (`Supervised`): low runs, medium asks,
high blocks."

The implemented path never looks at the HTTP method.
`ApprovalManager::approval_requirement`
(`crates/zeroclaw-runtime/src/approval/mod.rs:175-211`) checks, in order: `Full`
→ approved; `ReadOnly` → not required; `always_ask` → prompt; `auto_approve` →
approved; session allowlist → approved; otherwise prompt
(`approval/mod.rs:209-210`).

Consequence for Baixa: Solana JSON-RPC is POST, and under the default
`supervised` every `http_request` call prompts an operator regardless of method.
An unattended two-minute job has nobody to answer. Hence the explicit
`auto_approve` list in `config.example.toml`.

### 9.4 Whole config surfaces are absent from the repository

`docs/book/src/reference/config.md` and `reference/cli.md` are both linked from
`reference/index.md:5-6` and neither exists in the repo. The full config
reference is generated from the live schema at docs-build time and published at
`https://docs.zeroclawlabs.ai/master/en/reference/config.html`
(`README.md:110`).

Same for per-field tables inside pages: `channels/telegram.md:263` is literally
`{{#config-fields channels.telegram}}`, and `sop/syntax.md:344` is
`{{#sop-trigger-index}}`. The preprocessor lives at
`xtask/src/cmd/mdbook/peer_groups.rs:145`.

`sop/syntax.md:16-25` additionally declines on purpose to enumerate `SOP.toml`
manifest fields.

**Practical consequence:** reading a local clone's docs is not enough to write a
config or a SOP. Field names have to come from
`crates/zeroclaw-config/src/schema.rs` and
`crates/zeroclaw-runtime/src/sop/types.rs`. Every field in
`config.example.toml` and both `SOP.toml` files is cited to one of those.

### 9.5 `sops_dir` default: two answers

`docs/book/src/sop/syntax.md:3` says that when `sops_dir` is omitted, CLI
commands fall back to `<workspace>/sops` for offline inspection "but runtime SOP
execution is disabled." The schema doc comment
(`crates/zeroclaw-config/src/schema.rs:22558-22560`) says "the runtime and CLI
both resolve the default `<workspace>/sops`; SOPs load from there whenever it
exists."

Not resolved, because setting `sops_dir` explicitly makes it moot — and setting
it is what registers the `sop_*` tools anyway
(`docs/book/src/tools/overview.md:52`). `config.example.toml` sets it.

---

## 10. Smoke test

```
# In Telegram, to your bot:
invoice Acme Studio 250 USDC for August
```

Expect a compact reply with the `solana:` URL and a QR link.

Then:

1. `zeroclaw service logs --follow` and wait up to two minutes for a
   `SOP maintenance tick` line. Its attributes include `cron_started`,
   `cron_skipped`, and `cron_no_match` (`src/main.rs:7859-7871`). A
   `cron_started` of 1 means reconcile ran.
2. Pay the invoice from a wallet.
3. Within two minutes, expect `✅ Invoice #1 — Acme Studio · 250 USDC paid`
   with a solscan link.

Then work through INJECTION_TEST.md and record actual results against the
expected ones.

---

## 11. What this costs to run

Baixa's bill is dominated by one number: `reconcile` fires every two minutes,
which is **720 runs a day**. Each run is at least one model turn even when
nothing is open, because step 1 has to read the ledger before it can conclude
`nothing open`.

### A free provider will not run this agent, and the reason is arithmetic

Groq is the obvious candidate: free tier, no card, OpenAI-compatible, supported
by ZeroClaw at `[providers.models.groq.<alias>]`. Reproducibility is 15% of this
bounty's score, and a judge who can run the project without paying anyone is
worth more than one who cannot. It was tried properly. It does not work, for
three independent reasons found in this order.

**`openai/gpt-oss-120b` breaks the tool protocol.** Groq returns
`400 tool_choice is none, but model called a tool`, and the attempted call names
`tool.sop_status` rather than `sop_status`.

**`llama-3.3-70b-versatile` emits the wrong syntax.** It writes llama's *text*
function format into a native tool-calling request —
`<function=sop_execute={...}</function>` — which Groq rejects as
`tool_use_failed`. Setting `native_tools = false`, which is ZeroClaw's default
for Groq precisely because of this family's history, does not help.

**The free tier caps at 12,000 tokens per minute.** This agent's system prompt is
roughly **13,000 tokens per call**: every skill body inlined at
`prompt_injection_mode = "full"`, measured at 3.0M input tokens across 233 calls
in `state/costs.jsonl`. A single turn does not fit inside the per-minute quota.
The 429 is not a burst problem to back off from; the request is larger than the
budget it is drawn against.

That third point generalises past Groq. **Any free tier with a per-minute token
cap below ~15k cannot run a Tier 1 ZeroClaw agent that inlines its skills.** The
lever is `prompt_injection_mode = "compact"`, which keeps skill bodies out of the
prompt and loads them through `read_skill` on demand. It would fit — at the cost
of one mandatory extra tool call per turn, on models that had just demonstrated
they mangle ordinary ones. Not a trade worth making here; possibly the right one
for a smaller agent.

### The cost ceiling only covers models named in `[cost.rates.*]`

`enforcement.mode = "block"` refuses a request once `daily_limit_usd` is
reached. Reaching it requires the runtime to know what the model costs. **A
model absent from the rate sheet is priced at $0.00, so the limit is never
reached and the block never fires.**

Measured on this deployment. The sheet listed `claude-opus-5` and
`claude-haiku-4-5`. The provider model was switched to `claude-sonnet-4-5` and
no row was added:

| Model | Calls | Input tokens | Tracked | Real |
|---|---:|---:|---:|---:|
| `claude-sonnet-4-5` | 233 | 3,016,851 | $0.00 | **$9.28** |
| `claude-haiku-4-5` | 195 | 1,318,153 | $0.64 | $1.37 |
| `claude-opus-5` | 26 | 128,966 | $0.40 | $0.80 |
| | | | **$1.04** | **$11.44** |

The counter read $1.04 against a $3.00 ceiling while the real bill was over
eleven dollars. Nothing in `zeroclaw status` indicated a problem — a $0.00 model
looks identical to a cheap one.

**Add a row for every model you might switch to, before you switch to it.** The
example config now prices five. Audit your own spend from
`<workspace>/state/costs.jsonl`, which records per-call `input_tokens`,
`output_tokens`, and the runtime's `cost_usd` — comparing that last field
against the published rate card is how this was found.

The second number in that table matters as much as the first: 3.0M input tokens
across 233 calls is roughly 13,000 tokens per call. That is the system prompt,
with every skill body inlined at `prompt_injection_mode = "full"`, re-sent on
every call. At 720 reconcile runs a day the dominant cost is re-transmitting
instructions, not doing work.

### The lever that matters most: don't run it 24/7

The daemon is a foreground process you start and stop (`§7`). Nothing about
Baixa requires continuous uptime — an invoice that goes unreconciled for six
hours reconciles on the next run after you start the daemon, because status is
derived from the chain, not from having watched it happen. Chain state is not
missed, only observed later.

For a demo, an evaluation, or an operator who bills monthly, running the daemon
for a few hours a day costs single-digit dollars in total. Running it
continuously on an Opus-class model does not.

### Order-of-magnitude estimate

These are estimates from the run count and the prompt shape, not measurements.
Measure your own with `zeroclaw cost` before trusting them.

| Model | $/1M in | $/1M out | ≈ 1 hour, `*/2` | ≈ 24 h, `*/2` |
|---|---|---|---|---|
| `claude-opus-5` | 5 | 25 | $1.50–2.50 | $35–60 |
| `claude-sonnet-5` | 3 | 15 | $0.60–1.00 | $14–24 |
| `claude-haiku-4-5` | 1 | 5 | $0.30–0.50 | $7–12 |

Two things move these numbers down that are not in the table: prompt caching
(cache reads bill at roughly a tenth of input) and `[skills]
prompt_injection_mode = "compact"`, which keeps `SKILL.md` out of the system
prompt and loads it on demand. Whether ZeroClaw 0.8.3 sets `cache_control` on
Anthropic requests has not been verified here, so the estimates assume it does
not. `compact` is left off by default because the reliability of skill-following
matters more to this project than the saving.

### The ceiling is configured, not assumed

`[cost]` in `config.example.toml` sets `daily_limit_usd = 3.00` and
`enforcement.mode = "block"`. That second key is the point. ZeroClaw's default
enforcement mode is `warn` (`schema.rs:6521-6523`), which records the overage
and keeps spending — a limit that does not limit. `block` refuses the request
instead.

`allow_override = false` closes the `--override` escape hatch
(`schema.rs:6460-6463`).

The `[cost.rates.*]` sheet is filled in by hand so the ceiling is computed from
known list prices rather than a catalog lookup that may not carry the model.

### Other levers, in the order worth trying

1. **Stop the daemon when you are not watching.** Largest effect, no code change.
2. **Widen the cron.** `*/2` is tuned for a demo where a payment should close
   while the viewer is still watching. `*/10` cuts model calls fivefold and is
   more than fast enough for real invoicing. Edit
   `sops/reconcile/SOP.toml` and restart (`§8`).
3. **Change the model.** `[providers.models.anthropic.default] model`. Haiku 4.5
   is five times cheaper than Opus 5 and weaker at the multi-condition
   verification in reconcile step 3 — which is the step this whole project is
   about. Downgrade deliberately, not by default.
4. **`route_down` instead of `block`.** `[cost.enforcement] mode = "route_down"`
   with `route_down_model` set moves to a cheaper model at the ceiling rather
   than stopping (`schema.rs:6510-6515`). Untested here.
