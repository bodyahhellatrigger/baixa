# SPEC.md — ZeroClaw platform recon for Baixa

Stage 1 output. No code written. Every claim below cites `file:line` in the
ZeroClaw source. Anything I could not find in the source is marked **NOT FOUND**.

## 0. Source of truth

| Item | Value |
|---|---|
| Repository | `https://github.com/zeroclaw-labs/zeroclaw` (the only official repo, per `zeroclaw/README.md:187-191`) |
| Local clone | `~/baixa/zeroclaw` (shallow, default branch) |
| HEAD | `19de521a2ada7e6599f393581af058749863c95f`, 2026-08-02 |
| Docs read | `zeroclaw/README.md`, `zeroclaw/docs/book/src/**` |

Paths in citations are relative to `~/baixa/`, so `zeroclaw/README.md:105` means
line 105 of the cloned repo's README.

**Read this first:** large parts of the operator-facing config surface are
*generated at docs-build time* from the live Rust schema and do not exist as
prose in the repository. `zeroclaw/docs/book/src/channels/telegram.md:263` is
literally `{{#config-fields channels.telegram}}`; the preprocessor lives at
`zeroclaw/xtask/src/cmd/mdbook/peer_groups.rs:145`. So for exact field names I
cite the schema itself (`zeroclaw/crates/zeroclaw-config/src/schema.rs`), which
is the same source those pages render from.

---

## 1. Config file

### 1.1 Location and load order

One TOML file (`zeroclaw/README.md:105`). Resolution order is documented on the
struct: `ZEROCLAW_CONFIG_DIR` env → `ZEROCLAW_WORKSPACE` env → `~/.zeroclaw/config.toml`
(`zeroclaw/crates/zeroclaw-config/src/schema.rs:75`).

On-disk layout around it (`zeroclaw/docs/book/src/architecture/runtime-state-and-persistence.md:19-37`):

```text
<install>/
├── config.toml
├── .secret_key                 # key for encrypted secrets
├── data/
│   ├── sop/runs.db             # durable SOP run state
│   ├── cron/jobs.db
│   └── memory/
├── shared/                     # shared skill bundles live here
└── agents/<alias>/workspace/   # per-agent sandbox
```

`schema_version` is stamped on every write
(`zeroclaw/crates/zeroclaw-config/src/schema.rs:125-127`, written at `:21336`).

### 1.2 Section index (only sections Baixa needs)

Top-level `Config` struct starts at
`zeroclaw/crates/zeroclaw-config/src/schema.rs:78`. Relevant fields:

| TOML section | Shape | Schema line |
|---|---|---|
| `[providers.models.<type>.<alias>]` | nested provider registry | `schema.rs:135-137` |
| `[agents.<alias>]` | `HashMap<String, AliasedAgentConfig>` | `schema.rs:452-459` |
| `[risk_profiles.<alias>]` | `HashMap<String, RiskProfileConfig>` | `schema.rs:461-464` |
| `[runtime_profiles.<alias>]` | `HashMap<String, RuntimeProfileConfig>` | `schema.rs:466-469` |
| `[skill_bundles.<alias>]` | `HashMap<String, SkillBundleConfig>` | `schema.rs:471-474` |
| `[peer_groups.<name>]` | `HashMap<String, PeerGroupConfig>` | `schema.rs:486-494` |
| `[channels]` (and `[channels.telegram.<alias>]`) | `ChannelsConfig` | `schema.rs:284-287`, telegram map at `:13077` |
| `[memory]` | `MemoryConfig` | `schema.rs:289-292` |
| `[skills]` | `SkillsConfig` | `schema.rs:239-242` |
| `[sop]` | `SopConfig` | `schema.rs:641-645` |
| `[scheduler]` | `SchedulerConfig` | `schema.rs:220-224` |
| `[cron.<alias>]` | `HashMap<String, CronJobDecl>` | `schema.rs:269-276` |
| `[http_request]` | `HttpRequestConfig` | `schema.rs:372-376` |

### 1.3 Minimum viable config

`zeroclaw/README.md:112` states a V3 config needs at minimum four
`<type>.<alias>`-shaped section headers: a provider entry, an agent referencing
it, and a risk profile the agent gates against. It points at
`docs/book/src/providers/configuration.md#minimal-working-example` for "the
canonical four-section form with inline type/alias commentary".

That target section (`zeroclaw/docs/book/src/providers/configuration.md:5-7`)
contains **no TOML block** — only prose telling you to configure it through the
gateway, zerocode, or `zeroclaw config set`. So the canonical four-section
literal example is **NOT FOUND** in the repository.

A provider entry's shape is shown at `zeroclaw/README.md:126-130`:

```toml
[providers.models.openai.coding]   # type = openai; alias = coding (you choose)
model = "gpt-5.4"
wire_api = "responses"
requires_openai_auth = true
```

and an agent points at it with `model_provider = "openai.<alias>"`
(`zeroclaw/README.md:132`).

### 1.4 `[agents.<alias>]` fields Baixa will use

Struct at `zeroclaw/crates/zeroclaw-config/src/schema.rs:3540`:

| Field | Type | Line |
|---|---|---|
| `enabled` | `bool`, default `true` | `:3543-3544` |
| `channels` | `Vec<ChannelRef>`, e.g. `["telegram.home"]`; validation fails loud on dangling refs | `:3545-3550` |
| `model_provider` | dotted `"<type>.<alias>"` | `:3551-3556` |
| `risk_profile` | alias ref | `:3557-3560` |
| `runtime_profile` | alias ref | `:3561-3564` |
| `skill_bundles` | `Vec<String>` of bundle aliases | `:3565-3570` |
| `cron_jobs` | `Vec<String>` referencing `[cron.<alias>]`; this agent is the actor when the job fires | `:3598-3603` |
| `memory` | per-agent backend block `[agents.<alias>.memory]` | `:3703-3710` |
| `workspace` | per-agent sandbox block | `:3693-3701` |

### 1.5 `[risk_profiles.<alias>]` and approval

Struct at `zeroclaw/crates/zeroclaw-config/src/schema.rs:11772`:

| Field | Line |
|---|---|
| `level` (`AutonomyLevel`, default `supervised`) | `:11773-11774` |
| `auto_approve` — tools that never require approval | `:11788-11789` |
| `always_ask` — tools that always require approval | `:11790-11791` |
| `allowed_tools` — empty means no constraint | `:11812-11840` |
| `excluded_tools` — removed on non-CLI channels | `:11841-11847` |

`AutonomyLevel` variants are `ReadOnly`, `Supervised` (default), `Full`
(`zeroclaw/crates/zeroclaw-config/src/autonomy.rs:19-27`).

**The approval decision keys off the tool *name*, not the HTTP method or a risk
tier.** `ApprovalManager::approval_requirement`
(`zeroclaw/crates/zeroclaw-runtime/src/approval/mod.rs:175-211`) checks, in
order: `Full` → approved; `ReadOnly` → not required; `always_ask` → prompt;
`auto_approve` → approved; session allowlist → approved; otherwise **prompt**.
Under the default `supervised`, any tool not in `auto_approve` prompts an
operator.

### 1.6 Custom sections — a blocker for Baixa

Baixa's hard rule says the recipient address comes only from config. The
platform gives you no supported way to declare a custom top-level section.

- The root `Config` struct (`zeroclaw/crates/zeroclaw-config/src/schema.rs:76-78`)
  carries no `#[serde(flatten)]` catch-all and no `deny_unknown_fields`, so a
  hand-added `[baixa]` block **loads without error and is silently dropped from
  the in-memory struct**.
- `Config::save()` rebuilds the whole file by serializing the typed struct
  (`schema.rs:21370-21374`), so a full save **erases** any unknown section.
- `Config::save_dirty()` patches an existing file through `toml_edit`
  (`schema.rs:21454-21466`) and does preserve unknown keys — but it falls back
  to `save()` when the file is missing (`schema.rs:21418-21424`).
- `zeroclaw channel bind-telegram` calls the full `save()`
  (`zeroclaw/crates/zeroclaw-channels/src/orchestrator/mod.rs:7131`), and the
  in-Telegram `/bind` flow also saves config
  (`zeroclaw/docs/book/src/channels/telegram.md:177,181`).

So a `[baixa]` section survives until the first operator bind, then vanishes.
Separately, even while present, **no agent-callable tool can read arbitrary
config keys** — see §7. I have not picked a workaround; that is a Stage 2
decision I need from you (§9).

---

## 2. Skills

### 2.1 Canonical format: `SKILL.md`

`zeroclaw/crates/zeroclaw-runtime/src/skills/constants.rs:4` sets the canonical
manifest filename to `SKILL.md`. Line `:8` lists `SKILL.toml` and `manifest.toml`
as **deprecated** — "still accepted by the audit loader for back-compat with
installed skills. Never written by the service."

This contradicts `zeroclaw/docs/book/src/tools/skills.md:17`, which presents
`SKILL.md` and `SKILL.toml` as co-equal recommended local authoring formats.
**Write `SKILL.md`.**

The audit loader requires one of the three manifests at the skill root
(`zeroclaw/crates/zeroclaw-runtime/src/skills/audit.rs:60-65`). Optional scaffold
subdirs are `scripts/`, `references/`, `assets/` (`constants.rs:13`).

### 2.2 Frontmatter

The struct is `SkillFrontmatter`
(`zeroclaw/crates/zeroclaw-runtime/src/skills/frontmatter.rs:12-27`):

```markdown
---
name: create-invoice          # required; lowercase, hyphens only (frontmatter.rs:34-39)
description: ...              # required; third person; injected into the system prompt (frontmatter.rs:41-45)
license: MIT                  # optional, SPDX (frontmatter.rs:46-51)
author: ...                   # optional (frontmatter.rs:52-57)
version: 0.1.0                # optional, SemVer; 0.1.0 on scaffold (frontmatter.rs:58-63)
category: ops                 # optional (frontmatter.rs:64-69)
tags: [ops]                   # optional; the `slash` tag opts into slash commands (frontmatter.rs:70-84)
slash_options: []             # optional, typed (frontmatter.rs:25-26)
---

# Body — the instructions the agent follows.
```

`zeroclaw/docs/book/src/tools/skills.md:65` lists only `name`, `description`,
`version`, `author`, `tags`. The struct also accepts `license`, `category`, and
`slash_options`. Trust the struct.

Without frontmatter, the directory name becomes the skill name and the first
non-heading paragraph becomes the description
(`zeroclaw/docs/book/src/tools/skills.md:47`).

### 2.3 Where skills live and how an agent loads them

Three locations (`zeroclaw/docs/book/src/tools/skills.md:5-9`):

1. Per-agent workspace: `<install>/agents/<alias>/workspace/skills/<name>/`
2. Shared bundles: `<install>/shared/skills/<bundle>/<name>/`
3. Global: `<install>/data/skills/<name>/` — **agents do not load these automatically**

Bundles are the runtime mechanism. A bundle is `[skill_bundles.<alias>]`; when
`directory` is omitted it resolves to `<install>/shared/skills/<alias>/`
(`zeroclaw/docs/book/src/tools/skills.md:11`,
`zeroclaw/docs/book/src/architecture/runtime-state-and-persistence.md:50`).

`SkillBundleConfig` (`zeroclaw/crates/zeroclaw-config/src/schema.rs:12009-12016`):

```toml
[skill_bundles.baixa]
directory = "..."   # Option<String>, relative to workspace root (:12010-12011)
include = []        # empty = include all skills in `directory` (:12012-12013)
exclude = []        # (:12014-12015)
```

The agent must list the bundle in `agents.<alias>.skill_bundles`
(`schema.rs:3565-3570`, restated at `docs/book/src/tools/skills.md:219`).

**Tier-1 relevant:** `zeroclaw/docs/book/src/tools/skills.md:19` warns that
*packaged, signed* skill bundles ride the plugin system, which prebuilt release
binaries do not include — "on a stock binary, the shared bundles on this page are
the supported mechanism." Plain directory bundles are fine for Baixa.

### 2.4 `[skills]` config

`SkillsConfig` at `zeroclaw/crates/zeroclaw-config/src/schema.rs:6065`:

| Field | Default | Line |
|---|---|---|
| `open_skills_enabled` | `false` | `:6066-6069` |
| `open_skills_dir` | `None` → `$HOME/open-skills` | `:6070-6073` |
| `allow_scripts` | `false` — blocks `.sh`, `.bash`, `.ps1`, shebang files | `:6074-6077` |
| `registry_url` | `https://github.com/zeroclaw-labs/zeroclaw-skills` | `:6078-6081` |
| `prompt_injection_mode` | `full` (vs `compact`) | `:6087-6090` |
| `[skills.skill_creation]` | `enabled = false` | `:6091-6094`, struct at `:6110` |

Baixa needs none of these turned on. Leave `allow_scripts = false`.

### 2.5 Skill authoring CLI

```sh
zeroclaw skills bundle add <bundle>
zeroclaw skills add <name> --bundle <bundle> --description "..." --edit
zeroclaw skills list --agent <alias>     # what the agent actually loads
zeroclaw skills audit <name>
```

(`zeroclaw/docs/book/src/tools/skills.md:30-35,101-141`)

---

## 3. SOPs and cron triggers

### 3.1 Directory layout

```text
<workspace>/sops/
  reconcile/
    SOP.toml     # required
    SOP.md       # optional, but a run with no parsed steps fails validation
```

(`zeroclaw/docs/book/src/sop/syntax.md:5-14`,
`zeroclaw/docs/book/src/sop/how-it-works.md:5`)

### 3.2 `SOP.toml`

The docs deliberately refuse to enumerate manifest fields
(`zeroclaw/docs/book/src/sop/syntax.md:16-25`: "This page intentionally does not
enumerate manifest fields or provide hand-authored manifest examples"). The
parse target is `SopManifest`
(`zeroclaw/crates/zeroclaw-runtime/src/sop/types.rs:584-595`):

```
sop        -> SopMeta          (types.rs:585)
triggers   -> Vec<SopTrigger>  (types.rs:586-587)
positions  -> Vec<StepPosition> (Blueprint editor canvas coords; types.rs:588-592)
steps      -> Vec<SopStep>     (types.rs:593-594)
```

`SopMeta` (`zeroclaw/crates/zeroclaw-runtime/src/sop/types.rs:607-633`):

| Key | Required | Default | Line |
|---|---|---|---|
| `name` | yes | — | `:608` |
| `description` | yes | — | `:609` |
| `version` | no | `"0.1.0"` (`types.rs:668-670`) | `:610-611` |
| `priority` | no | `SopPriority::default()` | `:612-613` |
| `execution_mode` | no | falls back to `[sop] default_execution_mode` | `:614-615` |
| `cooldown_secs` | no | see `default_cooldown_secs` | `:616-617` |
| `max_concurrent` | no | `1` (`syntax.md:33`) | `:618-619` |
| `deterministic` | no | `false` — opt-in, no LLM round-trips between steps | `:620-622` |
| `admission_policy` | no | `parallel` | `:623-625` |
| `max_pending_approvals` | no | `0` = unlimited | `:626-628` |
| `agent` | no | parent agent alias that owns the procedure | `:629-632` |

Worked example of the `[sop]` table plus a trigger
(`zeroclaw/docs/book/src/sop/syntax.md:64-75`):

```toml
[sop]
name = "deploy-prod"
description = "Production deploy with approval"
version = "1.0.0"
max_concurrent = 1
admission_policy = "hold"
max_pending_approvals = 8

[[triggers]]
type = "manual"
```

### 3.3 Triggers

`SopTrigger` is a serde-tagged enum, `tag = "type"`, `rename_all = "lowercase"`
(`zeroclaw/crates/zeroclaw-runtime/src/sop/types.rs:126,142`). Variants relevant
to Baixa:

```toml
[[triggers]]
type = "cron"
expression = "*/2 * * * *"     # types.rs:161-164 — the ONLY field on this variant
```

```toml
[[triggers]]
type = "channel"
channel = "telegram"           # ChannelKind snake_case (types.rs:207-209)
alias = "home"                 # optional; unset matches every instance (types.rs:210-212)
condition = '$.foo == "bar"'   # optional (types.rs:213-215)
```

```toml
[[triggers]]
type = "manual"                # fired by the sop_execute tool (types.rs:217-218)
```

### 3.4 Does the cron trigger actually fire? Yes.

The documentation contradicts itself here, so I checked the code.

- `zeroclaw/docs/book/src/sop/fan-in/cron.md:3` says **Wired**.
- `zeroclaw/docs/book/src/reference/feature-matrix.md:64-67` says cron triggers
  are "defined and matched but not yet routed to a live source".

The code agrees with `cron.md`. `src/main.rs:7829` defines
`spawn_sop_maintenance`, which builds a `SopCronCache` (`main.rs:7839-7841`) and
loops on a ticker (`main.rs:7842-7847`) calling `run_sop_maintenance_tick`
(`main.rs:7899`). `SopCronCache::from_engine`
(`zeroclaw/crates/zeroclaw-runtime/src/sop/dispatch.rs:1233`) parses each SOP's
cron expression, and `check_sop_cron_triggers`
(`dispatch.rs:1311-1343`) dispatches a `SopEvent` for every expression whose
occurrence fell in the window `(last_check, now]`.

Treat `feature-matrix.md:64-67` as stale.

Details that matter for a 2-minute schedule:

- **5-field crontab is normalized to 6-field** by prepending a seconds field
  (`dispatch.rs:1252-1253`), so `*/2 * * * *` parses fine.
- **At-most-once per window.** If several occurrences fall inside one window,
  the SOP fires once, by design (`dispatch.rs:1325-1333`).
- **Poller, not a per-schedule timer.** Firing depends on the maintenance tick,
  whose period is `[sop] maintenance_interval_secs`, default `60`
  (`zeroclaw/crates/zeroclaw-config/src/schema.rs:22586-22593`). A 2-minute cron
  under a 60s tick is safe; a sub-minute cron would not be.
- **Requires the daemon.** The tick is spawned by `zeroclaw daemon` or the
  `zeroclaw channel start` supervisor, built with the `agent-runtime` feature
  and `sop.sops_dir` set. Standalone `zeroclaw gateway start` does not spawn it
  (`zeroclaw/docs/book/src/sop/fan-in/cron.md:3`).
- **Schedules are parsed once at startup.** A SOP added while the daemon runs
  needs a reload (`cron.md:3`).
- Setting `maintenance_interval_secs = 0` disables the tick entirely
  (`schema.rs:22589-22593`).

### 3.5 `SOP.md` step format

Steps are parsed from the `## Steps` section
(`zeroclaw/docs/book/src/sop/syntax.md:106-152`):

```md
## Steps

1. **Preflight** — Check service health.
   - tools: http_request
   - requires_confirmation: true
   - policy: prod
   - input: {"type":"object","required":["version"],"properties":{"version":{"type":"string"}}}
   - output: {"type":"object","required":["digest"],"properties":{"digest":{"type":"string"}}}
   - next: 3
```

Parser behavior (`zeroclaw/docs/book/src/sop/syntax.md:154-179`):

| Bullet | Meaning |
|---|---|
| `- tools:` | maps to `suggested_tools` |
| `- allow-tools:` / `- deny-tools:` | explicit per-step tool scope |
| `- requires_confirmation: true` | enforces approval for that step |
| `- kind:` | `execute` (default) or `checkpoint` |
| `- input:` / `- output:` | JSON-Schema-like boundary contracts |
| `- when:` | routing guard against accumulated completed-step outputs |
| `- next:` / `- depends_on:` | non-linear routing |
| `- on_failure:` | `fail`, `retry:<count>`, or `goto:<step>` |
| `- mode:` | overrides execution mode for that step |
| `- policy:` | names a `[sop.approval].policies` key |

Numbered items define step order; leading `**Bold**` becomes the title
(`syntax.md:156-157`).

Condition and `when:` grammar (`syntax.md:348-424`): a single comparison,
JSON-path form `$.path.to.field <op> <value>` or bare numeric. Operators
`>=`, `<=`, `!=`, `==`, `>`, `<`. **No `AND`/`OR`/`NOT`** (`syntax.md:423-424`).
JSON booleans compare as the quoted strings `"true"`/`"false"`
(`syntax.md:420-421`). Evaluation fails closed on invalid conditions, missing
payloads, and unresolved paths (`syntax.md:364-366`).

Step `when:` guards see accumulated outputs shaped as
`{"steps": {"1": {"severity": "critical"}}}` (`syntax.md:354-362`).

### 3.6 `[sop]` config

`SopConfig` at `zeroclaw/crates/zeroclaw-config/src/schema.rs:22557`:

| Field | Default | Line |
|---|---|---|
| `sops_dir` | `Option<String>`; omitted resolves to `<workspace>/sops` | `:22558-22562` |
| `default_execution_mode` | `supervised`; also `auto`, `step_by_step`, `priority_based`, `deterministic` | `:22564-22568` |
| `max_concurrent_total` | see `default_sop_max_concurrent_total` | `:22570-22572` |
| `approval_timeout_secs` | `0` disables the sweep | `:22574-22579` |
| `maintenance_interval_secs` | `60`; `0` disables | `:22586-22593` |
| `persist_runs` | `true` | `:22595-22601` |
| `run_store_backend` | `sqlite` | `:22603-22606` |
| `run_state_dir` | `<data_dir>/sop` | `:22608-22611` |
| `approval_mode` | `both` | `:22613-22617` |
| `approval_timeout_action` | `escalate` (fail-closed, never self-approves) | `:22619-22624` |
| `step_scope_enforce` | `false` — `tools:` stays advisory | `:22637-22639` |
| `step_schema_enforce` | `true` | `:22645-22647` |
| `untrusted_payload_max_bytes` | `8192` | `:22657-22660` |
| `untrusted_input_guard` | `warn` (also `block`, `sanitize`) | `:22662-22664` |

`sops_dir` has a **documentation contradiction**: `syntax.md:3` says that when
it is omitted, "CLI commands fall back to `<workspace>/sops` for offline
inspection, but runtime SOP execution is disabled", while the schema doc comment
(`schema.rs:22558-22560`) says "the runtime and CLI both resolve the default
`<workspace>/sops`; SOPs load from there whenever it exists." Set `sops_dir`
explicitly and the ambiguity disappears.

Setting `sops_dir` is also what registers the `sop_*` tools
(`zeroclaw/docs/book/src/tools/overview.md:52`).

### 3.7 SOP run mechanics

- Runs progress through the `sop_status`, `sop_approve`, `sop_advance` tools
  (`zeroclaw/docs/book/src/sop/how-it-works.md:8`).
- Run state persists to `<data_dir>/sop/runs.db` under
  `sop.persist_runs = true`; init failure logs a warning and falls back to
  process-local memory (`how-it-works.md:9`).
- **SOP audit records are persisted in the configured Memory backend under
  category `sop`** (`how-it-works.md:10`).
- Validate with `zeroclaw sop validate <name>`
  (`zeroclaw/docs/book/src/sop/syntax.md:434-436`). Validation warns on empty
  names/descriptions, missing triggers, missing steps, and step-numbering gaps
  (`syntax.md:441`).
- CLI manages definitions only — `list`, `validate`, `show`
  (`how-it-works.md:6`).

### 3.8 Alternative: declarative `[cron.<alias>]` agent jobs

A second, simpler scheduling path exists that does not involve the SOP engine.
`CronJobDecl` (`zeroclaw/crates/zeroclaw-config/src/schema.rs:12687`):

| Key | Notes | Line |
|---|---|---|
| `name` | optional human-readable | `:12688-12690` |
| `job_type` | `"shell"` (default) or `"agent"` | `:12691-12693` |
| `schedule` | tagged enum, see below | `:12694-12696` |
| `command` | required when `job_type = "shell"` | `:12697-12699` |
| `prompt` | required when `job_type = "agent"` | `:12700-12702` |
| `enabled` | default `true` | `:12703-12705` |
| `allowed_tools` | `Option<Vec<String>>`; strict explicit-list intersection | `:12709-12712` |
| `uses_memory` | default `true` | `:12713-12716` |
| `session_target` | `"isolated"` (default) or `"main"` | `:12717-12719` |
| `delivery` | see §7.2 | `:12720-12723` |

`CronScheduleDecl` (`schema.rs:12774-12785`), `tag = "kind"`, lowercase:

```toml
[cron.reconcile.schedule]
kind = "cron"
expr = "*/2 * * * *"
tz = "UTC"          # optional

# or: kind = "every", every_ms = 120000
# or: kind = "at",    at = "<RFC 3339>"
```

The agent that executes the job is the one listing the alias in
`agents.<alias>.cron_jobs` (`schema.rs:3598-3603`).

Trade-off for Baixa: `[cron]` gives you a free-form agent prompt with built-in
channel delivery and no daemon-tick indirection, but no steps, no approval
gates, no audited run state. The bounty framing ("SOP reconcile") points at the
SOP engine; the `[cron]` path is the fallback if the SOP tick proves flaky in
testing.

---

## 4. Wiring a Telegram channel

### 4.1 Three sources of truth

`zeroclaw/docs/book/src/channels/telegram.md:9-11` — the channel block owns the
connection, the agent block owns routing, peer groups own inbound authorization.

### 4.2 `[channels.telegram.<alias>]`

`TelegramConfig` at `zeroclaw/crates/zeroclaw-config/src/schema.rs:13741`:

| Field | Type / default | Line |
|---|---|---|
| `enabled` | `bool`, default `false` | `:13742-13748` |
| `bot_token` | `String`, `#[secret]`, encrypted at rest | `:13749-13760` |
| `api_base_url` | official Telegram endpoint by default | `:13761-13765` |
| `stream_mode` | `StreamMode` | `:13766-13769` |
| `draft_update_interval_ms` | raise this on `Too Many Requests` | `:13770-13773` |
| `debounce_ms` | `Option<u64>`, overrides `[channels].debounce_ms` | `:13774-13779` |
| `interrupt_on_new_message` | `bool` | `:13780-13784` |
| `mention_only` | `bool` — groups only; DMs always processed | `:13785-13789` |
| `ack_reactions` | `Option<bool>` | `:13790-13795` |
| `proxy_url` | `Option<String>` | `:13796-13800` |
| `approval_timeout_secs` | default 120s before auto-deny of an inline-keyboard approval | `:13801-13805` |
| `excluded_tools` | `Vec<String>` not exposed on this channel | `:13807-13811` |
| `reply_min_interval_secs` | outbound pacing floor, `0` disables | `:13812-13815` |
| `reply_queue_depth_max` | `0` → 16 when pacing is on; newest send dropped when full | `:13816-13822` |

Defaults are enumerated at `schema.rs:13825-13844`.

**There is no `allowed_users` field** (`telegram.md:29-31`). Authorization is
peer groups only.

### 4.3 Setup

```sh
zeroclaw config set channels.telegram.home.bot_token   # masked secret prompt
zeroclaw config set channels.telegram.home.enabled true
zeroclaw agents list
zeroclaw config set agents.primary.channels            # omit value to open the list editor
```

(`zeroclaw/docs/book/src/channels/telegram.md:52-64`)

Resulting non-secret structure (`telegram.md:68-75`):

```toml
[channels.telegram.home]
enabled = true
# bot_token is stored encrypted after the masked `config set` prompt

[agents.primary]
channels = ["telegram.home"]
```

**Binding rule** (`telegram.md:76-84`): once *any* agent declares a `channels`
list, a channel that is enabled but absent from an enabled agent's list is not
started. If no agent declares bindings, ZeroClaw falls back to legacy routing —
every enabled channel starts under the default enabled agent. Declare bindings
explicitly.

Never paste the token into `config.toml`, logs, or source control
(`telegram.md:41-42`).

### 4.4 Authorization

Two paths (`telegram.md:88-119`).

**One-time pairing.** Leave the resolved external-peer set empty; the channel
prints a one-time code to foreground output and structured logs, and the first
approved user redeems it with `/bind 123456` in Telegram
(`telegram.md:98-100,162`). On success ZeroClaw prefers the stable numeric
sender ID, writes it into `[peer_groups.telegram_home]`, and saves `config.toml`
(`telegram.md:180-183`). The running channel's peer resolver reads the shared
config, so no restart is needed.

**Pre-authorize** (`telegram.md:108-112`):

```toml
[peer_groups.telegram_home]
channel = "telegram.home"
external_peers = ["111111111", "222222222"]
```

Numeric IDs beat usernames because they survive a rename
(`telegram.md:104-106`). `channel = "telegram"` (type-wide) accepts the
identities on every Telegram alias (`telegram.md:114-116`).

`external_peers = ["*"]` accepts every sender who can reach the bot and disables
pairing — flagged CAUTION at `telegram.md:121-126`. Do not use it for Baixa.

Operator-side bind (`telegram.md:196-217`):

```sh
zeroclaw channel bind-telegram 111111111 --alias home
```

`--alias` must match the key in `[channels.telegram.<alias>]`; the CLI defaults
to `default` and rejects unknown aliases (`telegram.md:206-215`).

### 4.5 Running

```sh
zeroclaw daemon                # normal operation
zeroclaw channel start         # foreground diagnostic, starts all channels
zeroclaw channel doctor
zeroclaw service logs --follow
```

(`telegram.md:133-148`)

Telegram uses `getUpdates` long polling — no inbound port, no public callback
URL (`telegram.md:143-145`). Two processes sharing one bot token produce
`Telegram polling conflict (409)` (`telegram.md:257`).

Reload semantics (`telegram.md:221-228`): a direct `config.toml` edit or a
standalone `zeroclaw config set` takes effect only after a daemon reload or
process restart — saving alone does not rebuild long-running listeners.

### 4.6 Ships in the stock binary

`channel-telegram` is inside the `default-channels` bundle
(`zeroclaw/Cargo.toml:306-310`), which is in `default`
(`zeroclaw/Cargo.toml:282-289`) alongside `agent-runtime`.

Release artifacts build `--no-default-features` against the `dist` selection
(`zeroclaw/.github/workflows/release-stable-manual.yml:319,328`), and `dist`
resolves to the expansion of `default` plus `dist_extra_features`
(`zeroclaw/xtask/src/generate/spec.rs:673-686`), where `dist_extra_features` is
`channel-matrix`, `channel-lark`, `whatsapp-web`
(`zeroclaw/Cargo.toml:259-263`).

**`plugins-wasm` is not in `default` and not in `dist_extra_features`**
(`zeroclaw/Cargo.toml:394,427-429`). A stock release binary therefore carries no
WASM plugin host — Baixa's Tier-1 rule is satisfied by using the released binary
as-is.

---

## 5. `http_request` tool signature

### 5.1 Tool contract

`zeroclaw/crates/zeroclaw-tools/src/http_request.rs`:

- name: `"http_request"` (`:438-440`)
- description (`:442-445`): "Make HTTP requests to external APIs. Supports GET,
  POST, PUT, DELETE, PATCH, HEAD, OPTIONS methods. Security constraints:
  allowlist-only domains, local/private hosts blocked unless explicitly
  configured, configurable timeout and response size limits."

Parameters schema (`:447-475`):

| Param | Type | Required | Default | Line |
|---|---|---|---|---|
| `url` | string | **yes** | — | `:451-454` |
| `method` | string | no | `"GET"` | `:455-459` |
| `headers` | object | no | `{}` | `:460-464` |
| `auth_secret` | string | no | — | `:465-468` |
| `body` | string | no | — | `:469-472` |

`auth_secret` names a key in `[http_request.secrets]`, sent as the
`Authorization` header; entries may be literal, encrypted, or `${ENV_VAR}`, and
it overrides any literal `Authorization` header (`:467`).

Output schema (`:478-489`), all four fields required:

| Field | Type | Note |
|---|---|---|
| `status` | integer | HTTP status code |
| `reason` | string | canonical status reason |
| `headers` | string | **sensitive values redacted** |
| `body` | — | parsed JSON when the body is JSON, raw string otherwise |

Note `body` in the *response* is parsed JSON when possible. Shaping the response
down to ~200 tokens is Baixa's job, not the tool's.

### 5.2 `[http_request]` config

`HttpRequestConfig` at `zeroclaw/crates/zeroclaw-config/src/schema.rs:7473`:

| Field | Default | Line |
|---|---|---|
| `enabled` | `true` | `:7474-7476` |
| `allowed_domains` | `["*"]` — exact or subdomain match | `:7477-7479`, default at `:7524-7526` |
| `max_response_size` | `1_000_000` bytes; `0` = unlimited | `:7480-7482`, `:7516-7518` |
| `timeout_secs` | `30` | `:7483-7485`, `:7520-7522` |
| `allow_private_hosts` | `false` (SSRF protection) | `:7486-7489` |
| `allowed_private_hosts` | `[]`; `*` permits all private/local hosts | `:7490-7493` |
| `secrets` | `HashMap<String,String>`, `#[secret]`, encrypted | `:7494-7499` |

Full defaults at `:7502-7514`.

For Baixa: narrow `allowed_domains` to the Solana RPC host and
`api.qrserver.com`. Leaving `["*"]` is the shipped default and would not violate
any hard rule, but it widens the agent's reach for no benefit.

### 5.3 Approval consequence for Solana RPC

Solana JSON-RPC is `POST`. `zeroclaw/docs/book/src/tools/overview.md:89-93`
describes a low/medium/high risk model in which "`http_request` POST to
unconstrained URLs" is High and "Default (`Supervised`): low runs, medium asks,
high blocks."

The implemented approval path does not read the HTTP method at all — it matches
on tool name only (`zeroclaw/crates/zeroclaw-runtime/src/approval/mod.rs:175-211`,
detailed in §1.5). So the practical rule is simpler than the docs suggest:
under `supervised`, `http_request` prompts for every call regardless of method
unless the risk profile lists it in `auto_approve`.

An unattended 2-minute reconcile loop cannot answer prompts. Baixa's risk
profile will need `auto_approve = ["http_request", ...]` (keeping `level =
"supervised"`), or `level = "full"`. The narrower `auto_approve` route is
better: it leaves `shell` and everything else gated.

---

## 6. Reading and writing memory

### 6.1 Tools

All three are always registered
(`zeroclaw/docs/book/src/developing/tool-inventory.md:42`,
`zeroclaw/docs/book/src/tools/overview.md:32-33,43`).

**`memory_store`** (`zeroclaw/crates/zeroclaw-tools/src/memory_store.rs:23-50`):

| Param | Required | Line |
|---|---|---|
| `key` | **yes** — unique key | `:35-38` |
| `content` | **yes** — the information to remember | `:39-42` |
| `category` | no — `core` (permanent), `daily` (session), `conversation` (chat), or a custom name; defaults to `core` | `:43-46` |

`required: ["key", "content"]` (`:48`).

**`memory_recall`** (`zeroclaw/crates/zeroclaw-tools/src/memory_recall.rs:21-56`)
— no required params (`:29-56`, no `required` array):

| Param | Note | Line |
|---|---|---|
| `query` | omit or pass bare `*` to return recent memories | `:33-36` |
| `limit` | integer, default 5 | `:37-40` |
| `since` | RFC 3339; validated, invalid input returns an error | `:41-44`, validation `:64-74` |
| `until` | RFC 3339 | `:45-48`, validation `:75-77` |
| `search_mode` | `bm25` \| `embedding` \| `hybrid`; defaults to config | `:49-53` |

**`memory_forget`** (`zeroclaw/crates/zeroclaw-tools/src/memory_forget.rs:23-42`)
— `key` required (`:35-40`). Returns whether the memory was found and removed
(`:28`).

`memory_export` and `memory_purge` also exist
(`zeroclaw/docs/book/src/tools/overview.md:43`); I did not read their schemas
since Baixa does not need them.

### 6.2 Invoice-record implications

`memory_store` takes a flat `key` + `content` string. Baixa's invoice record
`{id, counterparty, amount_usdc, reference_pubkey, description, status, created_at,
paid_at, tx_signature}` has to be serialized into `content` (JSON string is the
obvious choice) under a key like `invoice:<id>`. Status transitions are a
re-store on the same key.

I did not find a documented upsert guarantee in the tool layer. The schema doc
for `core_retention_days` states that "neither recall nor ordinary rewrites
refresh `created_at` under the current SQLite upsert"
(`zeroclaw/crates/zeroclaw-config/src/schema.rs:10591`), which confirms that
same-key rewrites are upserts rather than duplicate rows. Stage 2 should verify
this behavior against a live store before relying on it.

Listing open invoices means `memory_recall` with a query, which is a **scored
relevance search**, not a `WHERE status = 'open'` filter. Under `hybrid` mode a
`min_relevance_score` of 0.4 drops low scorers
(`schema.rs:10621-10625`). Enumerating open invoices reliably will need either a
single index record holding the open-ID list, or `search_mode = "bm25"` with a
distinctive key prefix. This is a Stage 2 design decision I am flagging, not
resolving.

### 6.3 `[memory]` config

`MemoryConfig` at `zeroclaw/crates/zeroclaw-config/src/schema.rs:10559`:

| Field | Default | Line |
|---|---|---|
| `backend` | `<backend>.<alias>`; bare `"sqlite"` → `"sqlite.default"`; `"none"` disables persistence | `:10560-10566` |
| `auto_save` | saves what *you* tell ZeroClaw as conversation history; the agent's own replies are not saved | `:10567-10569` |
| `hygiene_enabled` | periodic archive/retention pass | `:10570-10572` |
| `archive_after_days` / `purge_after_days` | | `:10579-10584` |
| `core_retention_days` | `0` = keep forever; absolute age from first write | `:10591-10593` |
| `embedding_provider` | `none` = keyword-only, no API calls | `:10594-10596` |
| `search_mode` | `bm25` \| `embedding` \| `hybrid` | `:10618-10620` |
| `min_relevance_score` | `0.4` | `:10621-10625` |

Per-agent override lives at `[agents.<alias>.memory]`; the `backend` field is
locked at agent creation and immutable afterward
(`schema.rs:3703-3710`).

Memory rows are agent-scoped, and same-backend cross-agent recall is opt-in
(`zeroclaw/docs/book/src/architecture/runtime-state-and-persistence.md:51`).

Set `core_retention_days = 0` so invoice records never age out.

---

## 7. Sending a message to a channel

There is no general "post arbitrary text to channel X" tool. Four narrower paths
exist.

### 7.1 `send_message_to_peer` (agent-callable)

`zeroclaw/crates/zeroclaw-runtime/src/tools/send_message_to_peer.rs:41-68`:

| Param | Required | Line |
|---|---|---|
| `channel` | **yes** — channel ref, e.g. `"telegram.prod"`; must be one of the agent's configured channels *and* one the target peer also listens on | `:53-56` |
| `target` | **yes** — a peer agent's alias or an external peer's username, e.g. `"@operator"` | `:57-60` |
| `message` | **yes** | `:61-64` |

The tool is bound to one calling agent's alias and validates every send against
that agent's resolved peer set (`:18-25`, resolver imported at `:9`). Delivery
goes through `deliver_announcement` (`:8`, called at `:226`).

**Constraint:** `target` must resolve inside the peer set. It is not a free-form
chat ID.

### 7.2 Cron job delivery (`announce`)

`DeliveryConfigDecl` (`zeroclaw/crates/zeroclaw-config/src/schema.rs:12803-12816`):

| Field | Note | Line |
|---|---|---|
| `mode` | `"none"` or `"announce"` | `:12804-12806` |
| `channel` | e.g. `"telegram"`, `"discord"` | `:12807-12809` |
| `to` | target/recipient identifier | `:12810-12812` |
| `thread_id` | optional; required by channels routing on a separate thread field | `:12813-12816` |

Worked form from the `cron_add` tool description
(`zeroclaw/crates/zeroclaw-runtime/src/tools/cron_add.rs:170`):

```
delivery={"mode":"announce","channel":"discord","to":"<channel_id_or_chat_id>"}
```

This is the path that accepts a raw chat ID.

### 7.3 `deliver_announcement` (the shared outbound primitive)

`zeroclaw/crates/zeroclaw-runtime/src/cron/scheduler.rs:1062-1089`, signature
`(config, channel, target, thread_id, output)`. It dispatches through a
`DELIVERY_FN` registered by the binary. **When no handler is registered it logs
a warning and returns `Ok(())`** (`:1079-1088`) — sends silently no-op outside
the daemon. Plan Baixa's testing around a real `zeroclaw daemon`.

### 7.4 SOP approval routes

A `[sop.approval.policies.<name>]` may carry `request_route` (delivered when a
run parks at a gate) and `escalation_route` (delivered only on timeout), both
shaped `channel:recipient` where channel is `<channel>.<alias>` or bare
`<channel>` for a singleton (`zeroclaw/docs/book/src/sop/syntax.md:186-203`).
Delivery is best-effort, never blocks or clears the gate, has no durable retry
queue, and fires only in the daemon (`syntax.md:199-206`).

### 7.5 What does *not* send to a channel

`sessions_send` (`zeroclaw/crates/zeroclaw-tools/src/sessions.rs:326-348`)
appends a message to a session's conversation history as a `user` message. It is
inter-agent plumbing, not an outbound channel send.

`channel_room` (`zeroclaw/crates/zeroclaw-tools/src/channel_room.rs:67-122`)
creates and manages rooms — `action` and `channel` required. Not a message send.

### 7.6 The ordinary reply path

For Baixa's `create_invoice`, the agent replying to the operator's Telegram
message needs none of the above. The channel orchestrator delivers the agent's
turn output back to the originating chat automatically — that is the standard
loop shown at `zeroclaw/docs/book/src/channels/telegram.md:13-21`. The tools in
§7.1–7.4 matter only for the *unattended* reconcile notification, where no
inbound message is being answered.

---

## 8. NOT FOUND

| Item | Status |
|---|---|
| `docs/book/src/reference/config.md` — the full generated config reference | **NOT FOUND** in the repo. Linked from `reference/index.md:6` and, as an external URL, from `README.md:110` (`https://docs.zeroclawlabs.ai/master/en/reference/config.html`). It is generated from the live schema at docs-build time. |
| `docs/book/src/reference/cli.md` | **NOT FOUND**. Linked from `reference/index.md:5`. |
| Canonical four-section minimal `config.toml` example | **NOT FOUND**. `README.md:112` points at `providers/configuration.md#minimal-working-example`; that section (`configuration.md:5-7`) has no TOML block. |
| Per-channel Telegram field table | **NOT FOUND** as prose. `telegram.md:263` is the placeholder `{{#config-fields channels.telegram}}`. Fields taken from `schema.rs:13741-13823` instead. |
| SOP trigger field tables | **NOT FOUND** as prose. `syntax.md:344` is `{{#sop-trigger-index}}` and `fan-in/cron.md:9` is `{{#sop-trigger cron}}`. Fields taken from `sop/types.rs:142-218`. |
| Hand-authored `SOP.toml` manifest example | Deliberately withheld — `syntax.md:16-25`. Fields taken from `sop/types.rs:584-633`. |
| A supported way to declare a custom top-level config section | **NOT FOUND**. See §1.6. |
| An agent-callable tool that reads an arbitrary config key | **NOT FOUND**. The nearest surfaces are `model_routing_config` and `proxy_config` (`tool-inventory.md:47`), both scoped to their own subsystems. |
| Anything Solana, USDC, Solana Pay, ed25519 keygen, or QR | **NOT FOUND**, as expected. Baixa supplies all of it. |
| An explicit token/character cap on tool responses | **NOT FOUND** as a tool-level contract. The nearest knob is the orchestrator's `max_tool_result_chars` (`zeroclaw/crates/zeroclaw-channels/src/orchestrator/mod.rs:495`), which truncates rather than shapes. The ~200-token shaping is Baixa's own discipline inside skill and SOP instructions. |

---

## 9. Findings that change Stage 2

Ordered by how much they affect the build.

**1. The recipient address cannot live in a custom config section.**
Unknown sections load but are dropped from the typed struct, and the first
`zeroclaw channel bind-telegram` or Telegram `/bind` erases them via the full
`save()` path (§1.6). Even before that, no tool can read them. Before I write
any code I need your call on where `recipient` and `usdc_mint` live. Options I
can see, none of which I have committed to:

  - a `[skill_bundles.baixa] directory` pointing at a directory whose `SKILL.md`
    body carries the constants — config-controlled path, file-controlled value;
  - `[http_request].secrets` entries — a typed, encrypted `HashMap<String,String>`
    (`schema.rs:7494-7499`), though semantically it is for `Authorization` values;
  - a `[cron.<alias>].prompt` string carrying the constants, if we take the
    §3.8 cron path instead of the SOP path.

Each bends the hard rule differently. Say which one you want and I will build it.

**2. `http_request` needs `auto_approve` or the reconcile loop deadlocks.**
Under the default `supervised` autonomy, every `http_request` call prompts an
operator (§1.5, §5.3). An unattended 2-minute job has nobody to answer. My
recommendation: keep `level = "supervised"` and set
`auto_approve = ["http_request", "memory_store", "memory_recall"]` on Baixa's
risk profile.

**3. The SOP cron trigger works, but the documentation says it does not.**
Code confirms firing (§3.4). Do not be alarmed by
`reference/feature-matrix.md:64-67` during review. Practical constraints: the
daemon must be running, `sop.sops_dir` must be set, schedules parse once at
startup, and a 2-minute expression sits comfortably above the 60-second tick.

**4. `memory_recall` is relevance search, not a query.**
Enumerating open invoices needs either an index record or a bm25 key-prefix
convention (§6.2). I will propose one at Stage 2 unless you have a preference.

**5. Write `SKILL.md`, not `SKILL.toml`.**
The code marks `SKILL.toml` deprecated and never writes it, even though the docs
still recommend it (§2.1).

**6. Tier 1 holds.**
The stock release binary ships `agent-runtime` and `channel-telegram` and does
**not** ship `plugins-wasm` (§4.6). Skills-as-directories, SOPs, config, and
`http_request` are all available on the released artifact with zero plugins and
zero WASM.

**7. Bounty scope worth re-checking.**
The Superteam Brasil listing is titled "Build Solana-native plugins for
Zeroclaw". Baixa's hard rules forbid plugins. If the listing means WASM plugins
in the `zeroclaw-plugins` sense, a Tier-1 skills-and-SOPs build may not meet the
acceptance criteria. Your call, but confirm before Stage 2.
