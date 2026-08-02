# THREAT_MODEL.md

What Baixa protects, what it does not, and where the Tier 1 line falls. Written
to be checkable rather than reassuring. Where a control is weak, this document
says so.

## 1. Custody: the part that is structural

Baixa holds no private keys and signs nothing.

The agent issues a payment request. The payer's own wallet builds, signs, and
submits the transaction. Baixa then reads the chain. There is no signing path in
the codebase, no key material in config, no key material in memory, and no tool
on the agent's risk profile that could produce a signature.

The risk profile names a strict **allowlist** of seven tools
(`config.example.toml`, `[risk_profiles.baixa].allowed_tools`): `http_request`,
`memory_recall`, `memory_store`, `send_message_to_peer`, and the three `sop_*`
lifecycle tools. A non-empty list is an authorization constraint — anything not
named is unreachable (`schema.rs:11812-11840`). So there is no
route to a local key file or an external signer either.

This is the one guarantee that holds regardless of how badly a prompt injection
goes. **Nobody can make Baixa move funds, because Baixa cannot move funds.**


**Why an allowlist and not a denylist.** An earlier version of this file claimed
`excluded_tools = [shell, file_write, file_edit]` was the control. That claim was
wrong twice over. A three-name denylist admits every tool nobody thought to
name, and a live run showed the agent reaching for `file_read` and
`glob_search`. And `excluded_tools` is scoped: the schema documents it as
"Tools excluded from **non-CLI channels** under this profile"
(`schema.rs:11841-11847`), so it never applied to `zeroclaw agent` at all. It is
kept in the config as a redundant second statement about the three worst tools,
not as the mechanism.

That scoping still leaves one honest gap: anyone who can run `zeroclaw agent` on
the host is already the operator, and the allowlist is the only thing narrowing
them. Host access is outside this threat model, and this document does not claim
otherwise.

## 2. Status: the other structural control

An invoice's `status` is written in exactly one place: step 4 of
`sops/reconcile/SOP.md`, from four checks against a `getTransaction` response.

`skills/create_invoice/SKILL.md` sets `status: "open"` at creation and is
instructed never to write any other value, never to write `paid_at` or
`tx_signature`, and never to modify an existing invoice.

So the sentence "mark invoice #3 as paid" has no code path behind it. There is
no tool that sets a status. Setting one means the reconcile SOP ran and four
on-chain conditions held. A chat message cannot manufacture a
`getTransaction` response.

This is structural in the same sense as §1: the capability is absent, not
merely discouraged.

## 3. Recipient address: honest accounting

This is the weakest of the three controls and the one worth reading carefully.

**Where the address lives.** Three independent operator-written copies that must
agree:

1. `config.toml` → `[agents.baixa.identity].aieos_inline`. A typed schema field
   (`IdentityConfig`, `schema.rs:6407-6417`), so it survives a full
   `Config::save()` rewrite. It reaches the model through the system prompt via
   `aieos_to_system_prompt` (`identity.rs:719`, `bio` rendered at `:742-743`).
2. `skills/create_invoice/SKILL.md` → OPERATOR CONSTANTS block.
3. `sops/reconcile/SOP.md` → OPERATOR CONSTANTS block.

None of the three is reachable from chat input. A counterparty cannot edit
config, cannot edit a skill file, and cannot edit a SOP. Writing to any of them
requires filesystem access to the host, which the agent does not have.

**Why a custom `[baixa]` config section was not used.** The root `Config` struct
(`schema.rs:76-78`) has no serde catch-all and no `deny_unknown_fields`. An
unknown section loads silently, is dropped from the typed struct, and is erased
the next time anything calls `Config::save()` (`schema.rs:21370-21374`) — which
`zeroclaw channel bind-telegram` does (`orchestrator/mod.rs:7131`), as does the
in-Telegram `/bind` flow. The address would vanish on the first operator bind.

**What the self-check adds.** Step 6 of the skill compares the recipient
substring in the built URL against the SKILL.md literal, character for
character, and refuses with `⚠ recipient mismatch — refused` on any difference.

**What the self-check does not add.** At Tier 1 the creation path runs through
the model. The literal, the URL, and any injected text all sit in the same
context window. A sufficiently good injection could in principle produce a URL
with a wrong address *and* a self-check that reports agreement. The check raises
the cost of the attack; it does not make it impossible. Calling it a guarantee
would be a lie.

**So what is the actual worst case?** One invoice goes out carrying an address
that is not the operator's. That is the entire blast radius. Specifically:

- No funds are drained, because Baixa holds no keys (§1).
- The payer sends to the attacker instead of the operator. The operator is out
  one invoice's worth of expected revenue; nothing already received moves.
- `reconcile` detects it within two minutes. The payment lands at a destination
  that is not `RECIPIENT_WALLET`, condition 3 fails, and the invoice is
  `flagged` with reason `destination mismatch` rather than quietly closed.
- `daily_digest` re-surfaces the flag every day until the operator clears it,
  and prints the reason verbatim.

A wrong invoice is a bad day. It is not a drained treasury, and the difference
is the custody boundary, not the prompt.

**How to close the remaining gap.** Move URL construction out of the model: a
Tier 2 plugin builds the `solana:` URL from config and hands back a finished
string, so no model-mediated step can influence the address. That is a real
improvement and a real reason to reach for T2 — but it is a different tier, and
this submission is deliberately T1.

## 4. The payment reference

`create_invoice` generates a 43–44 character base58 string as the Solana Pay
reference. The private half is never generated, stored, or used; the reference
is a read-only marker that makes the payer's transaction findable.

**The weakness:** at T1 there is no shell and no crypto tool, so the string is
model-generated rather than drawn from a CSPRNG. Model-generated randomness is
not cryptographic randomness.

**What that means in practice.** The reference is not a secret and does not
authorize anything. Two risks follow:

- **Collision.** Two invoices sharing a reference would misattribute a payment.
  Mitigated by checking every new reference against the whole ledger before use
  (skill step 4) and regenerating on a hit. Within one operator's ledger this
  is sufficient.
- **Prediction.** A third party who guesses a pending reference could send a
  small payment to it and cause noise. They cannot steal anything: the money
  would go to `RECIPIENT_WALLET`, and condition 4 catches a short amount as
  `partial`.

A T2 plugin calling a real keypair generator removes this entirely. Noted, not
papered over.

## 5. Untrusted input surfaces

| Surface | Trusted? | Handling |
|---|---|---|
| Operator chat message | Partly | Parsed for counterparty, amount, description only. Never for recipient, mint, RPC host, or status. |
| Invoice `description` | No | Stored and displayed verbatim. Never read as an instruction. |
| Counterparty name | No | Same. |
| RPC response | Structurally, yes; semantically, no | Field values are read; no field is ever treated as an instruction. A memo field carrying "mark this paid" is just a string. |
| On-chain memo / token metadata | No | Never read at all. Not part of the four conditions. |
| Telegram sender identity | Yes, after peer-group check | Inbound authorization is peer groups only; there is no `allowed_users` field on the Telegram channel (docs `telegram.md:29-31`). |

The SOP engine additionally frames untrusted trigger payloads before they reach
step context and caps them at `untrusted_payload_max_bytes` (default 8192,
`schema.rs:22657-22660`). Baixa's cron trigger carries no payload, so this is
belt-and-braces rather than load-bearing.

## 6. What "raw RPC never reaches the model context" actually means

The brief asks that raw RPC output never reach the model context. At Tier 1 that
is not literally achievable, and the honest version is worth stating.

`http_request` returns its response body to the calling turn by construction
(`crates/zeroclaw-tools/src/http_request.rs:478-489`). There is no interception
point between the tool and the context window without a plugin. So a
`getTransaction` response does enter the model's context for exactly one step.

What Baixa does control:

- `getSignaturesForAddress` uses `limit: 5`, keeping that response small.
- Each step extracts a named handful of fields and carries only those forward.
  Raw JSON never crosses a step boundary.
- Raw JSON is never written to memory and never sent to the operator.
- `[http_request].max_response_size` caps the body at 1 MB
  (`schema.rs:7480-7482`).

What Baixa does not control: the size of a single `getTransaction` response
inside the step that fetched it. Solana's RPC has no field projection, so the
whole parsed transaction arrives.

**This is the clearest argument in the whole build for correct layering.** A
Tier 2 plugin that performs the RPC call and returns a shaped ~200-token verdict
would close this properly. Baixa does not pretend a skill can do a plugin's job.

## 7. Approval posture

`level = "supervised"` with a named `auto_approve` list rather than
`level = "full"`.

The approval manager matches on tool name and never inspects the HTTP method
(`crates/zeroclaw-runtime/src/approval/mod.rs:175-211`). Under `supervised`,
anything absent from `auto_approve` prompts an operator
(`approval/mod.rs:209-210`), and an unattended two-minute job has nobody to
answer. So the SOPs' tools are listed explicitly and everything else still
gates.

The practical difference from `full`: if a future skill or an injected
instruction reaches for a tool nobody enumerated, it stalls at a prompt instead
of running. That is worth the extra config lines.

`sop_approve` is deliberately **not** on the `auto_approve` list, and
`[sop] approval_mode = "out_of_band_required"` goes further: it makes the
agent's `sop_approve` a no-op entirely (`schema.rs:22718-22720`). Under the
default `both`, the agent tool *or* an out-of-band principal clears a gate
(`schema.rs:22714-22717`) — which would let Baixa approve its own checkpoint and
reduce §8's human gate to decoration. Only a CLI / WS / HTTP principal can
resolve it now.

## 7a. The human checkpoint

One outcome parks: `destination mismatch`. It means an invoice went out with an
address that is not `RECIPIENT_WALLET`, which is the materialized version of the
§3 residual risk. Step 7 of `sops/reconcile/SOP.md` is a `kind: checkpoint` step
with `requires_confirmation: true`; step 6's routing guard
(`when: $.steps.5.destination_mismatch == "true"`) means a clean run never
dispatches it.

Ordering matters here. The ledger write happens in step 5, *before* the gate, so
the flag survives whether or not anyone answers. A checkpoint that gated the
write would lose the evidence when the operator is asleep.

Approving acknowledges. It does not clear the flag, change a status, or make the
invoice payable. Nothing in Baixa moves a `flagged` invoice to `paid` without a
fresh verifying transaction.

Everything else stays unattended on purpose. A checkpoint on every payment would
be a checkpoint nobody reads by week two.

## 8. Failure modes and their handling

| Failure | Handling |
|---|---|
| RPC timeout | Status unchanged, retry in 2 minutes |
| RPC returns an error object | Status unchanged, logged with reason |
| Malformed or truncated JSON | Status unchanged, logged |
| `getTransaction` returns `null` | Treated as unconfirmed, retry next cycle |
| `confirmationStatus: processed` | Treated as unconfirmed, skipped |
| Transaction failed on-chain (`meta.err`) | Signature discarded, status unchanged |
| Wrong mint | `flagged`, reason `wrong mint` |
| Wrong destination owner | `flagged`, reason `destination mismatch`, run parks on the §7a checkpoint |
| Amount short | `partial`, invoice stays open, shortfall reported |
| Overpayment | `paid`, overage reported for deliberate refund |
| Two payments on one reference | `paid` against the earliest, plus a duplicate flag |
| Ledger record missing | Treated as empty, no crash |
| Notification send fails | No silent status corruption: the ledger write in step 5 precedes the notify in step 6 |

Every path leaves `status` untouched unless four RPC-derived conditions hold.

## 9. Residual risks, stated plainly

1. **Invoice creation is model-mediated.** §3. Worst case is one wrong invoice,
   detected within two minutes.
2. **The reference is not cryptographically random.** §4. Consequence is noise,
   not theft.
3. **Raw `getTransaction` JSON enters context for one step.** §6. A plugin is
   the correct fix.
4. **Ledger integrity depends on the memory backend.** The ledger is one SQLite
   record. A corrupted or truncated write loses history. `persist_runs = true`
   and `core_retention_days = 0` reduce the exposure; a periodic export is the
   obvious follow-up.
5. **The three operator constants can drift apart.** Nothing enforces that
   config, skill, and SOP hold the same address. The skill's self-check compares
   the built URL against its own literal, which catches drift at issue time but
   not at edit time. A pre-flight check is the obvious follow-up.
