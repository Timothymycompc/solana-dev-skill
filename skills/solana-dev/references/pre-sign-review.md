# Pre-sign review: what to check before an agent signs

"Simulate before sending" (SKILL.md, W009) catches transactions that fail. It does not catch transactions that
succeed and hand over the user's funds: a drainer's approval simulates cleanly. This reference lists what to read
from a transaction and its simulation before asking for a signature, and when to refuse.

## Contents
- [Rules](#rules)
- [1. Resolve every account](#1-resolve-every-account)
- [2. Read the instructions, including CPIs](#2-read-the-instructions-including-cpis)
- [3. Simulate and diff balances](#3-simulate-and-diff-balances)
- [4. Lifetime: blockhash, durable nonce, already landed](#4-lifetime-blockhash-durable-nonce-already-landed)
- [5. Does it do what the user asked?](#5-does-it-do-what-the-user-asked)
- [Summary to show the user](#summary-to-show-the-user)

## Rules

- **Fail closed.** If any check can't complete (RPC error, a lookup table that won't resolve, a simulation
  error), don't sign. "Couldn't check" is not "safe".
- **Judge the bytes you will sign**, not a description of them. Re-run the review if the transaction is rebuilt.
- **A clean review is not a guarantee.** It means none of the known risks below were found.

## 1. Resolve every account

v0 transactions can load accounts from address lookup tables, so the static account keys don't show everything
the transaction touches. Fetch each table (`getAddressLookupTable` / `fetchAddressLookupTable`) and expand the
writable and read-only indexes before reviewing. If a table can't be fetched, refuse. v1 transactions don't
support lookup tables: every account is in the inline list ([transactions-v1.md](transactions-v1.md)).

Note the required signers (the first `numRequiredSignatures` keys). If the user's wallet is a signer it didn't
expect to be, or the fee payer isn't who the user thinks, stop.

## 2. Read the instructions, including CPIs

Decode every top-level instruction, and the inner instructions from the simulation (`innerInstructions: true`).
These patterns move control of funds without moving funds, so the simulation's balances look harmless:

| Program | Instruction (first data byte) | Risk |
|---|---|---|
| SPL Token / Token-2022 | `Approve` (4), `ApproveChecked` (13) to a delegate that isn't the user | The delegate can move up to the approved amount at any time. The most common drainer pattern. |
| SPL Token / Token-2022 | `SetAuthority` (6) | Account owner or close authority changes hands; whoever holds it controls the account. |
| SPL Token / Token-2022 | `CloseAccount` (9) with a destination that isn't the user | The account's rent deposit goes to someone else. |
| System | `Assign` (1, u32 LE) on the user's wallet | The wallet becomes owned by another program and stops being a normal wallet. |
| Stake / BPF Upgradeable Loader / nonce | authority changes (`Authorize`, `SetAuthority`, `AuthorizeNonceAccount`) | Control of the stake account, program or nonce passes to another key. |

Also flag any program not on a list the user or app trusts. An unknown program isn't proof of harm, but it is
where these patterns hide.

## 3. Simulate and diff balances

Simulate with a fresh blockhash and ask for the post-state of the accounts the user owns:

```json
{
  "jsonrpc": "2.0", "id": 1, "method": "simulateTransaction",
  "params": ["<base64 tx>", {
    "encoding": "base64", "sigVerify": false, "replaceRecentBlockhash": true, "innerInstructions": true,
    "accounts": { "encoding": "base64", "addresses": ["<user wallet>", "<user token accounts>"] }
  }]
}
```

Read the pre-state with `getMultipleAccounts` and diff lamports and token amounts (parse token accounts, or use
`jsonParsed`). The two reads are at different slots (`context.slot`), so a busy account can show a change the
transaction didn't cause; compare only accounts the transaction writes, and re-read if the numbers don't add up.
Report every outflow: SOL beyond the fee, and every token amount leaving the user.

## 4. Lifetime: blockhash, durable nonce, already landed

- **Expired blockhash:** `isBlockhashValid` returns `false` once the blockhash is older than ~150 blocks. The
  network will reject it; rebuild instead of signing.
- **Durable nonce:** if the first instruction is System `AdvanceNonceAccount` (4), the transaction never expires.
  A signed copy stays valid until the nonce advances, so whoever holds it can submit it later. Sign only if the
  user meant to.
- **Already landed:** for a transaction that already carries a signature, `getSignatureStatuses` with
  `searchTransactionHistory: true` on its first signature. If it's on chain, signing again does nothing useful
  and may hide a replay.

## 5. Does it do what the user asked?

Agents get amounts and addresses wrong more often than they get attacked. Compare the effect from step 3 to the
request: recipient, mint, and amount in base units, using the mint's real `decimals` from chain
(`getTokenSupply` or the mint account), never a remembered or guessed value. "Send 1 USDC" is 1_000_000 base
units; a transfer of 10_000_000 is a mismatch to refuse, even though nothing malicious is involved.

## Summary to show the user

Before asking for the signature (W009), show: cluster, fee payer, every outflow from step 3 with token symbols and
human amounts, any finding from steps 2 and 4, and whether the effect matches the request. Wait for an explicit
yes.

Hosted option (third-party, MIT-licensed): [Solana Pulse](https://github.com/Timothymycompc/solana-pulse-ai-agent-gateway)
runs steps 1-4 as `check_tx` (OK / REVIEW / BLOCK: lookup tables, token approvals, `SetAuthority`, `Assign`,
closes to others, unknown programs, simulation and outflows, blockhash, already landed; not yet durable nonces or
stake/loader/nonce authority changes) and step 5 as `tx_matches`, over HTTP or MCP.
