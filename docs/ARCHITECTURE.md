# VeilPayouts Architecture

> Draft v0 — will be revised by the STRK20 integration plan (`STRK20_INTEGRATION_PLAN.md`). Sprint scope: business payouts with claim links, on STRK20 mainnet.

## Goals

1. **Pay anyone by link** — recipients need no wallet address and no prior registration; the payout is escrowed behind a secret and claimed via a link.
2. **Private by default** — payout amount, payer, payee, and the payer↔payee link never appear on-chain.
3. **Shielded by design** — the payout lands in the recipient's shielded STRK20 balance as an encrypted note.
4. **Ship on mainnet** — three real STRK20 pool transactions, a live demo, and a 3-minute video by Aug 31.

## Components

### 1. Frontend (Next.js 16, TypeScript)

Forked from the [STRK20 starter kit](https://github.com/Akashneelesh/strk20-starter-kit): wallet picker (get-starknet v6, `eip1193Adapters: []`), `WalletAccountV6` connection, `strk20InvokeTransaction` action plumbing, `strk20Balances` reads.

Two faces, one app:

- **Business dashboard** (`/dashboard`): create a payout (amount → claim link), list payouts with status (pending / claimed / expired), refund unclaimed payouts.
- **Recipient claim** (`/claim/[token]`): opens the claim link, connects a wallet, registers if new, claims the payout into a shielded balance, shows the shielded balance landing.

### 2. STRK20 pool (money layer)

| Action | STRK20 action | Visibility |
|---|---|---|
| Fund escrow | `deposit` (shield) then `transfer` (private) | deposit public; transfer private |
| Claim payout | `transfer` ("OPEN" open note) + `invoke` (escrow `privacy_invoke` Claim) | open-note amount public; owner hidden |
| Cash out | `withdraw` (unshield) | public destination + amount |

### 3. PayoutEscrow (Cairo contract — the anonymizer)

A **stateful** `privacy_invoke` helper, adapted from the [STRK20-by-example escrow](https://strk20-by-example.org/helpers/escrow) worked example and extended for the business use case:

```
storage:
  privacy_contract: ContractAddress   // pinned at deploy; only the pool drives us
  commitments: Map<felt252, CommitmentEntry>

CommitmentEntry { token, amount, claimed, expires_at }

privacy_invoke(operation, commitment_hash, token, amount, secret, note_id):
  assert caller == privacy_contract

  Deposit:
    store CommitmentEntry{token, amount, claimed: false, expires_at}
    return []            // tokens stay parked in escrow
  Claim:
    hash = poseidon(ESCROW_COMMITMENT_TAG, secret)   // recompute; ignore arg
    entry = commitments[hash]; assert exists && !claimed
    mark claimed; approve pool to pull; 
    return [OpenNoteDeposit{note_id, token, amount}]  // credits claimer's note
  Refund:
    (only the payout creator / after expiry) return the deposit to the pool
```

**Sprint-scope decision:** the *create → share → claim* loop is the MVP. Refund/expiry is included as a second operation in the same contract if time allows; otherwise refunds are handled by re-claiming with the creator's own key material (documented honestly). See the integration plan for the phased breakdown.

### 4. Claim-link flow

1. Creator shields STRK (public deposit) — done ahead of time, unlinkable to any specific payout.
2. Creator builds a STRK20 transaction: `transfer` of the payout amount to the escrow (private), plus `invoke` on the escrow with `Deposit` + a random `secret`. The secret is generated client-side; only `poseidon(secret)` goes on-chain.
3. The claim link encodes: escrow address + payout id (the commitment hash) + the secret, signed/authenticated off-chain by the creator's app. Any bearer of the link can claim — links are shareable by design (that's the product).
4. Recipient opens the link, connects/registers, and the app submits `transfer` ("OPEN" → open note) + `invoke` (escrow `Claim` with the secret). The escrow verifies the preimage, and the pool credits the open note — the payout lands shielded.

## Mainnet targets

| Value | Address |
|---|---|
| STRK20 pool | `0x040337b1af3c663e86e333bab5a4b28da8d4652a15a69beee2b677776ffe812a` |
| Chain | `SN_MAIN` |
| RPC | `https://rpc.starknet.lava.build` (public) / Alchemy |
| PayoutEscrow (to deploy) | TBD — recorded in `strk20.json` |

## Non-goals (this sprint)

- Recurring/scheduled payouts, multi-token batches, CSV uploads, payroll runbooks.
- Server-side custody of any key material — the escrow secret lives in the claim link, protected by the app's transport (HTTPS) and the fact that it's spendable only by the escrow's `privacy_invoke` (pool-gated).
- A literal REST API product — the "API" is the claim-link rail + a dapp dashboard, keeping the demo self-contained.
