**Baixa — a USDC invoice agent that holds no keys**
ZeroClaw + Solana. Tier 1: stock binary, zero plugins, zero WASM.

Type `invoice Acme 250 USDC for August` in Telegram. It issues a Solana Pay link and QR, polls the chain every two minutes, and closes the invoice when the payment verifies.

**Custody T1.** Never holds a key, never signs. It builds the URL; the payer signs. Full compromise costs you one wrong invoice, not your balance.

Status is written in one place, after four conditions hold: no on-chain error, the mint matches, the destination **owner** matches (from `postTokenBalances`, not the token account), and the amount covers the invoice. No chat message can set a status: nothing has a tool that writes one. Four states, not two. A destination mismatch parks on a checkpoint.

**What is and isn't demonstrated.** Invoice issue is real, verified from the runtime trace: the recipient in the emitted URL matches the config constant character for character. Reconcile runs seven steps against a live invoice and leaves the status untouched. **No real payment has been reconciled.** Two of five injection vectors have live transcripts; the rest are predictions.

**The part I'd like reviewed.** Thirteen defects documented, each with its source line. All found by running it, and each passed `doctor`, `sop validate` and `skills list` while broken.

• A SOP cron trigger fires and nothing executes the run. Only `deterministic` runs reach a terminal state, so it fires once and never again.
• A skill can be installed, listed, and never reach the agent: `include` matches the skill's `name:`, not its directory. One hyphen, and `skills list` doesn't apply that filter.
• The ledger vanishes from its own store: the audit logger writes each step *and its tool outputs* into the same memory, so a search for the ledger finds the search instead.

Three of the thirteen correct my own earlier claims, kept as corrections.

https://github.com/bodyahhellatrigger/baixa
Tell me where the threat model is wrong.
