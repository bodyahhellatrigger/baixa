Why this bounty exists
ZeroClaw is a self-hosted AI agent runtime in Rust: 32k stars, one binary, 70+ model provider integrations, 30+ channels (Telegram, WhatsApp, Discord, Matrix, email, webhooks), an SOP engine with cron scheduling, channel triggers, and human approval checkpoints, skills, persistent memory, MCP client support, and GPIO on Raspberry Pi and ESP32. 

Its thesis: you own the agent, the data, and the machine it runs on. This bounty is about what that ownership makes possible when the agent can touch Solana.

What does an AI agent you actually own has to do with a blockchain?
A payment terminal living in a family shop's WhatsApp. A DeFi guardian that wakes you only when your position needs you. A $40 Raspberry Pi reporting sensor readings on-chain. An agent that hires another agent and settles in escrow, spawns scoped sub-agents for a job, earns behind its own machine paywall, or hunts bugs for hash-committed bounties. Nobody has drawn the map here — that's the point. 

Build a real use case on ZeroClaw and Solana, run it, and show us.

Self-hosted, privacy-preserving, lean, fast, Rust-native 🦀

We are judging use cases, not components. A plugin nobody uses is not a submission. A use case someone runs every day is.

What a submission is
A submission is a showcase post. There is no other submission format.

1. A working use case: a real ZeroClaw agent, on a real channel, doing a real job that involves Solana. Yours, running. Not a concept. Link to Github repo.

2. A showcase post in the #solana-bounty channel on the ZeroClaw Discord:

A video, 3 minutes or less: real agent, real channel, your use case doing the thing. No slides. Terminal + phone is perfect.

A write-up: what it does, who it's for, which ZeroClaw features it uses, what (if anything) you had to build, its custody tier and threat model, and links to your config/SOPs/skills/code so another operator could reproduce it. Secrets redacted.

Any supporting material that communicates your submission's quality and reliability.  

We are not accepting just standalone plugin PRs as submissions. If your use case needed a plugin, link the code from your showcase. Registry merges happen separately after judging: maintainers invite the strongest implementation per plugin family. Don't open registry PRs during the bounty.

Reproducibility is part of the submission. The best showcases end with another operator saying "I set this up in an evening."

Depth beats breadth: one use case someone runs every day beats five that demo once. Originality counts double out here — build something nobody's seen and actually run it, and you're not competing with the field, you're defining it. Brazil-first flows (PIX and USDC reconciliation, BRL invoicing) are especially welcome.

What a winning showcase looks like
Video: a phone screen. Someone DMs the shop's WhatsApp: "charge table 4, 25 USDC." The agent replies with a QR. A customer wallet pays it. Forty seconds later the agent posts "Invoice #412 paid ✓" in the owner's channel. Cut to the terminal showing the SOP run and the reference-key poll.

Write-up: stock release binary, webhook channel + WhatsApp channel, one payment skill (Solana Pay URL construction + response shaping), one cron SOP polling getSignaturesForAddress, one approval checkpoint for refunds. Custody tier T1: no keys held. Config gist attached, prompt-injection transcript included: a customer message tries to talk the agent into "refunding" to an attacker address; the checkpoint catches it.

That's the bar: boring infrastructure, composed well, provably running.

Three ways to build, least code first
Verified against current ZeroClaw master. Start at tier 1; go deeper only when the use case demands it. Correct layering is scored — a tier 1 solution to a tier 1 problem beats unnecessary WASM.

Tier 1 — Stock release, zero plugins (works today)
The release binary already covers most T0/T1 ideas:

- Built-in http_request and web_fetch tools, ON by default (public hosts allowed; private hosts blocked for SSRF safety). Your agent reaches any Solana RPC, any DAS endpoint (Helius, Triton, QuickNode, Alchemy), or Jupiter's REST API out of the box.

- Skills teach the workflows. Solana Pay transfer-request URLs are plain strings (solana:<recipient>?amount=...&spl-token=...&reference=...); payment detection is one RPC call (getSignaturesForAddress on the reference key). Jupiter's Swap V2 (api.jup.ag) returns a ready-to-sign base64 transaction over plain HTTPS, with a documented keyless tier (0.5 requests/second) for light usage and a free API key one step up. A well-written skill turns the stock agent into a payment terminal or a swap-preparer with zero compiled code.

- Skills can even live on-chain. A community experiment worth knowing: gitlana (https://github.com/tonbistudio/gitlana) packages agent skills as Metaplex Core assets — manifest and code entirely inside a Solana account, resolvable and sha256-verified from a single getAccountInfo call, with a live mainnet package. Experimental and brand new, but it points at something real: Solana as the distribution rail for agent capability itself. Safety note if you build on this frontier: on-chain verification proves integrity, not trustworthiness — a skill is model-visible text, so review anything before your agent ingests it.

- Blinks (Solana Actions) make on-chain actions nearly free at T1. An Action is an HTTP endpoint (GET returns a preview, POST returns a ready-to-sign base64 transaction); a Blink is a shareable URL wrapping it. Your agent drops the Blink in chat and the recipient's wallet builds, previews, and signs — zero key handling, zero SDK. Dialect stewards the public registry, and a use case can host its own Action endpoint.

- SOPs own schedules and approvals: cron triggers are wired into the daemon, inbound webhook-channel messages can start procedures via channel triggers, and per-step human approval checkpoints are real — a run pauses until someone approves. Watch-loops are cron-polling SOPs.

- Memory keeps addresses, thresholds, and preferences across sessions.

Tier 2 — Stock release + an existing MCP server (works today)
ZeroClaw is an MCP client (stdio/SSE/HTTP, configured per agent). Mature Solana MCP servers exist now — Helius MCP (queries, transactions, webhooks, wallet analysis) and SendAI's solana-mcp (60+ actions) — and plugging one in gives deep Solana capability with no compiled code. 

Services run by other agents are appearing on this surface too: capability registries where agents discover and hire each other with escrowed settlement, and intel or risk oracles run by agents for agents, on subscription.

Custody honesty required either way: an external MCP server that holds keys or signs transactions is a third party you're trusting — and a service run by someone else's agent doubly so. Declare it in your tier and threat model.

Tier 3 — Build a plugin (source-built host)
Some things genuinely need bounded code inside the sandbox: Token-2022 TLV parsing for risk checks, hand-built unsigned transactions, x402 settlement under a hard-coded cap, attestation signing. This is the self-hosted thesis at full strength — no trusted third-party server, deterministic code with declared permissions, auditable by anyone. When your use case needs it:

- A plugin is a WebAssembly component (wasm32-wasip2) implementing the tool-plugin world from wit/v0. One component = one tool. permissions = ["http_client", "config_read"] gives you outbound HTTPS (TLS performed host-side) and your own config section, with secrets decrypted from encrypted-at-rest storage. Sockets and websockets don't exist for plugins today; stay on HTTP.

- Follow the reference layout (plugins/redact-text): pure Rust core + thin #[cfg(target_family = "wasm")] shim, host-run tests with mocked RPC (no live network in tests), structured logging via the logging import, a manifest that declares only what you use, MIT license.

- Keep the code in your own repo or fork during the bounty and link it from your showcase.

- Machine commerce cuts both ways here. x402 is live on Solana at real scale: your agent can pay per-request for data and APIs (the facilitator co-signs as fee payer, so the agent needs no SOL for gas) — or sell its own capability behind a machine paywall and let other agents pay it per call — and where agents transact, agents also compete: arenas with on-chain prizes are already a thing. The 402 handshake is young enough that even its mechanics are open territory: tiered price menus in the 402 response, negotiated in one round trip, are already being explored. Hard per-day caps enforced in code are mandatory in either direction.

- Running it: plugins are not in the release binaries. Build the host from source — cargo build --release --features plugins-wasm-cranelift, set plugins.enabled = true, add your plugin's config entry. Judges score against exactly this bar. If a plugins-preview build gets published during the bounty, we'll pin it in #solana-bounty.

- The dependency reality (verified by build): the modular solana-pubkey / solana-instruction / solana-message / solana-transaction / solana-hash crates, plus borsh and bs58, all compile clean to wasm32-wasip2 on the stock toolchain — no hand-rolled byte encoding needed to build and serialize transactions. (Even solana-sdk itself compiles for wasip2 now; prefer the modular crates for a minimal component.) Two caveats: this is compile-verified as a library, not yet exercised as an instantiated component inside the ZeroClaw host, whose WASI capability grants are narrower — budget for surprises at the component boundary, and write down what you hit. And the browser-targeted crates (wasm_client_solana, solana-client-wasm) still won't work (JavaScript glue). Transport is unchanged either way: RPC goes over waki (blocking wasi:http) + serde_json, not solana-client.

- Shared infrastructure is a valid builder's use case: a clean wasm32-wasip2 Solana core crate (a thin waki JSON-RPC client, durable-nonce helpers, and convenience layers over the modular solana crates) in its own repo, proven by a real plugin and a showcase demonstrating it, competes for a top prize.

The custody ladder (non-negotiable)
An agent with key access and an LLM in the loop is a hot wallet with a prompt-injection surface. Every showcase declares its tier and defends it:

T0 Read. RPC reads, lookups, alerts. Secrets held: an RPC key at most.

T1 Build. Unsigned transactions, Solana Pay URLs, multisig proposals; a human or the host signs. Secrets held: none.

T2 Sign. Signs and submits. Only acceptable with hard spend caps and a mint allowlist enforced in code, a session key holding limited funds (never a main wallet), and an approval gate — ZeroClaw's SOP checkpoints exist for exactly this. If a judge can prompt-inject your agent into misusing funds, you score zero on safety regardless of how good everything else is.

T0 and T1 are the sweet spot and where most of the prize money will land.

Best practices/tips
Chain-native caps exist now: the audited Subscriptions & Allowances program (mainnet June 2026 — Allowances, Recurring Delegations, Subscription Plans) puts spend limits on-chain. A T2 design should prefer it to hand-rolled caps wherever it fits.

The strongest pattern: the agent proposes, a Squads multisig disposes. This is real today — Squads v4 has a Proposer role, so an agent member can create proposals it can never execute, and humans approve from SquadsX or the Fuse mobile app. Fair warning: building Squads proposals from a wasm component means hand-encoding Anchor-style instructions and deriving PDAs — the hardest encoding job in this bounty. Get your use case working end to end with a simpler approval path first; add Squads last.

And the custody design space is still wide open. The experimental edge is actively inventing here, and building one of these patterns into your use case counts as craft: policy wallets where the key lives behind a local daemon the model can never read, with spend limits and destination rules enforced outside the prompt. Transaction firewalls sitting between the agent and broadcast. Reputation-gated signing. Fail-closed action certification, where nothing leaves the machine unless the exact serialized transaction has been verified against intent. Privacy as an installable capability — stealth addresses, hidden amounts, compliance viewing keys. And underneath it all, on-chain agent identity is becoming real infrastructure: agents with registered identities and peer-earned reputation, content signed at creation so provenance survives reposting (the official Agent Registry is in Resources).

If your use case touches funds, include a prompt-injection test in your write-up: a malicious message tries to make the agent move funds it shouldn't, and your setup fails closed. Transcript required.

Judging
The use case — 30%. Is this a job someone actually needs done? Are YOU running it? Would a stranger set it up and still be running it in a month?

Safety & custody design — 25%. Tier honest, fails closed, prompt-injection tested when funds are involved, third-party trust (MCP servers, facilitators) declared.

Craft — 20%. Where you built code: pure core, real tests, idiomatic Rust. Where you composed: clean SOPs, sensible config, well-written skills. Correct layering counts.

Reproducibility — 15%. Could another operator replicate your setup from your write-up in an evening?

Showcase — 10%. Can we understand it in three minutes and run it in five?

Tiebreak: build-in-public logs on X during the bounty.

The traps
1. Blockhash expiry. A transaction waits in an approval queue while the human is at lunch; ~90 seconds later its blockhash is dead. This is the structural problem of approval-gated agent payments. Durable nonces solve it, with three gotchas: the nonce account locks ~0.0015 SOL of rent, AdvanceNonceAccount must be the first instruction, and one nonce account serializes to one in-flight transaction — parallel pending approvals need a nonce account each. Solving this well is worth points.

2. The wasm dependency wall turned out to be a door. The modular solana crates compile clean to wasm32-wasip2 (verified by build — see Tier 3). The remaining risk lives at the component boundary (wit-bindgen integration and the host's capability grants), not in the compiler. Budget time there, and document what you hit — the write-up is worth points.

3. Don't flood the context window. A raw getProgramAccounts response will nuke your agent's context and cost the operator real money on every call. Shape output to the ~200 tokens the model needs — in your plugin's response shaping or your skill's instructions. Judges will look at what your tools return.

4. wit/v0 is experimental — no .frozen marker, the ABI can move. Pin your assumptions and expect a rebuild.

5. RPC keys live in config, never in code. Read them via config_read (config secrets are encrypted at rest and handed to you decrypted). Support user-supplied RPC URLs — people run their own.

6. Pyth Core deprecates July 31, 2026 — mid-bounty. Unauthenticated Hermes endpoints stop serving; get a Pyth API key before you build, or use Switchboard as the fallback: its public Crossbar endpoint serves feed values over plain REST, unauthenticated but rate-limited and best-effort — fine for cron alerts, self-host Crossbar for anything production-grade. Do not demo on an endpoint that dies before judging.

7. Design for polling, not webhooks. A chat-resident agent on cron has no guaranteed public inbound ingress. Scheduled REST pulls of parsed transaction history beat webhook pushes here — and where building a transaction yourself is the hard part, route the action through a Blink instead (the protocols that hand back base64 transactions over REST — Actions, Jupiter, Drift Gateway, Kamino — are your T1 friends).

Rewards
🥇 1st: 1,800 USDG

🥈 2nd: 1,200 USDG

🥉 3rd: 1,000 USDG

Honorable mentions (x4): 250 USDG each

Total: 5,000 USDG

We will not accept
- Concepts, mockups, or slideware. The agent must run.

- A plugin with no use case around it. Components are not submissions here.

- Thin single-RPC-call wrappers padded into WASM — that's a skill plus the built-in http tool (see Tier 1).

- Anything holding a raw private key with no caps, no allowlist, and no approval gate. Instant disqualification.

- Trading bots, sniper bots, "buy this token" agents. This bounty is about giving agents safe hands.

Resources
ZeroClaw: repo · docs · Discord

Plugin authoring guide (start here): docs/book/src/plugins/

Reference plugin: plugins/redact-text

The WIT contract (read the actual .wit files, they are the spec): wit/v0

A published HTTP plugin to copy patterns from: plugins/telegram

Solana: Solana Pay spec · Token-2022 extensions · DAS API

Questions → #solana-bounty in the Zeroclaw Discord.



ZeroClaw is MIT/Apache-2.0 and community-maintained. The only official repository is github.com/zeroclaw-labs/zeroclaw. This bounty is sponsored by Superteam Brasil; upstream merge decisions belong to the ZeroClaw maintainers, and we'd rather you win them over than route around them.
