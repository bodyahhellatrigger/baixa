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
| `<SOLANA_RPC_HOST>` | RPC hostname, no scheme (e.g. `api.mainnet-beta.solana.com`) |
| `<OPERATOR_TELEGRAM_NUMERIC_ID>` | your numeric Telegram user ID |
| `<ABSOLUTE_PATH_TO>/sops` | absolute path, e.g. `/home/you/.zeroclaw/workspace/sops` |

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
