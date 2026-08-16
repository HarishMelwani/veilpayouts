# STRK20 Privacy Integration Plan — VeilPayouts

Generated 2026-08-16 by the strk20-privacy-integration skill. Statuses current at generation time — re-verify the "coming soon" items before building against them.

## 1. Project snapshot

- **Stack:** Next.js 16 / React 19 / TypeScript, `starknet@10.4.0` (STRK20-era `WalletAccountV6`), `@starknet-io/get-starknet-discovery@6.0.2` + `@starknet-io/get-starknet-wallet-standard@6.0.2`, zustand. Cairo via Scarb (`cairo/`), forked from the STRK20 starter kit.
- **Relevant code:**
  - `src/app/components/client/WalletHandle/SelectWallet.tsx` — get-starknet v6 discovery + `WalletAccountV6.connect`, wallet picker
  - `src/app/components/client/WalletHandle/WalletAccountV6Tag.tsx` — all STRK20 actions (`deposit` / `transfer` / `withdraw` / `invoke`), `strk20Balances`, helper deploy, receipt rendering
  - `src/utils/constants.ts` — token, RPC providers (Alchemy mainnet/sepolia), helper addresses, class hash
  - `cairo/src/lib.cairo` — starter's echo helper (`StrkInvokeHelper`) — pattern base for our escrow
  - `src/app/page.tsx` — single-page UI (starter layout)
- **Privacy goal (from interview):** hide who pays whom and the payout amount in business payouts; let a payer fund a recipient who has **not yet registered** a viewing key (escrow behind a secret + claim link); keep the payer↔payee link off-chain. Recipient collects into a shielded balance.
- **Environment:** testnet (Sepolia) first; mainnet for the sprint's required three pool transactions. Wallets: Ready today, Xverse when its dapp-facing Wallet API lands.

## 2. Chosen route: Privacy Wallet API (starknet.js) + app-specific anonymizer contract

VeilPayouts is a dapp whose users hold their own wallets, so all money movement goes through the user's wallet via `strk20InvokeTransaction` — the dapp never touches viewing keys. The escrow step (fund behind a secret, claim into a note) is a protocol action no wallet can do alone, so it needs our own stateful `privacy_invoke` helper (`PayoutEscrow`). Mixed route, buildable now: the Wallet API part is live, and the anonymizer is our own Cairo to write, review, audit, and deploy.

**The rule this follows:** this app never touches viewing keys — the user's wallet acts on its behalf via starknet.js. The escrow secret lives in the claim link (off-chain), never in the app or on-chain.

## 3. What this delivers — hidden vs visible

| Private | Public |
|---|---|
| Payout amount (escrow funding is a private transfer) | Shield deposits (address, token, amount — screened by FPI) |
| Payer identity (observers see pool → escrow, not who initiated) | Open-note amount at claim time (plaintext by design) |
| Payee identity (claim credits a shielded note only the claimer can open) | Fact + timing of pool interactions |
| Payer↔payee link (nothing on-chain connects creator to claimer) | Recipient's unshield, if they cash out |
| The escrow secret / claim link (off-chain bearer secret) | — |

Honest limits: the claim amount is public via the open note (STRK20 design — measured at execution); deposits and withdrawals are public legs. Claim identity privacy, never amount privacy for open notes or the public legs. Per the route: the anonymizer hides the *user's address*; the app-side amounts may still be public.

## 4. Prerequisites & versions

- `starknet@10.4.0` (pinned — npm `next` tag carries STRK20; `latest` 10.0.x has none of it). Installed.
- `@starknet-io/get-starknet-discovery` / `get-starknet-wallet-standard` — installed at 6.0.2; **drift noted**: next is 6.0.4 (re-verify pin at build time).
- `@starknet-io/types-js@0.10.3` — installed.
- Test wallet: Ready extension.
- Cairo: Scarb + Starknet Foundry (`snforge`/`sncast`) for the escrow contract; audit before mainnet.

## 5. Phase 1 — first shielded flow (starter plumbing is already wired)

1. Keep the starter's connect flow (`SelectWallet.tsx`) — get-starknet v6, `eip1193Adapters: []`, `WalletAccountV6.connect`.
2. Keep shield / private transfer / unshield actions (`WalletAccountV6Tag.tsx`); they are the app's money primitives.
3. Graceful degradation: the starter gates STRK20 actions to Mainnet/Sepolia and detects wallet capability via `supportedSpecs` — keep both; add capability detection via `supportedWalletApi` version query (never `strk20Balances` for feature detection — it prompts the user for data we don't need).
4. Verify against the Ready extension + the wallet test dapp (https://starknet-wallet-account.vercel.app/).
5. Day-0 mainnet proof: register + shield + private transfer + unshield on the live pool, record hashes in `strk20.json`.

## 6. Phase 2 — feature integration (the payouts product)

- **Create-payout flow** (`src/app/`): pick amount → generate a random secret client-side → build STRK20 actions `[transfer(amount, escrow), invoke(escrow, Deposit)]` → present claim link (escrow address + commitment hash + secret). UX labels the escrow as a pattern illustration; our contract is our own code (see Phase 3).
- **Claim-link landing** (`/claim/[id]`): connect/register wallet → actions `[transfer("OPEN", claimer), invoke(escrow, Claim, secret)]` → payout lands as a shielded note.
- **Shielded-balance display**: `strk20Balances` is a deliberate, planned feature on the recipient's claim-confirmation screen only (shows their own balance — the one allowed use).
- **Dashboard**: list payouts with status (pending/claimed/expired) from off-chain app state. If any per-user activity is ever read on-chain, it must read the pool's `Deposit` event filtered on its first indexed key — never the transaction sender (private transactions are relayed; the sender is the relayer for every user).
- **UX copy** follows the privacy model: shield deposits and unshields labeled public; open-note amounts labeled public; never claim deposit/withdrawal privacy.

## 7. Phase 3 — PayoutEscrow anonymizer contract (our own Cairo — tracked)

- **Design-now (this sprint):** two-operation state machine in `cairo/src/` (replacing the echo helper): `Deposit` (store `CommitmentEntry{token, amount, claimed, expires_at}` keyed by domain-separated `poseidon(ESCROW_COMMITMENT_TAG, secret)`, return empty span — funds parked), `Claim` (verify preimage, mark claimed, approve pool, return `OpenNoteDeposit`). Caller-gated to the pool address. Double-claim reverts.
- **Entry criterion:** our design review completes; the reference patterns are public (`packages/ekubo_swap_anonymizer`, `packages/vesu_lending_anonymizer` in the SDK monorepo — skeletons to adapt, not drop-ins). The by-example escrow page is an **unofficial pattern illustration only** — never a shipped package; our contract is our own code.
- **Audit step (non-negotiable, owner: harishmelwani):** review the contract before mainnet deployment; record the address in `strk20.json` `contracts`.
- Stretch (if time): expiry + refund operations in the same contract.
- The skill never generates Cairo — this is team code, and we are the team.

## 8. Testing

- Testnet-first sequence: deploy `PayoutEscrow` on Sepolia; fund → claim → unshield round-trip with two Ready-wallet accounts; verify atomicity (claim reverts roll back cleanly, no stranded tokens).
- `snforge` unit tests for the escrow (commit/claim/double-claim/expiry logic).
- Pure-local devnet does not exercise the wallet/proving path — final verification is on Sepolia against Ready + the wallet test dapp, then the three mainnet transactions.

## 9. Compliance & security notes

- Deposit screening is enforced onchain by the protocol (v0.14.3+); it applies on every route — self-hosted proving does not bypass it. Never present VeilPayouts as a screening workaround.
- Selective disclosure exists for legitimate regulatory requests; it is not automatic compliance and carries no regulator endorsement. We own our legal/compliance decisions.
- We own review, audit, deployment, and maintenance of `PayoutEscrow`. The claim link is a bearer secret — treat it like cash; a leaked link lets anyone claim.

## 10. Open items to re-verify at build time

- get-starknet 6.0.2 → 6.0.4 drift; `starknet` next is 10.7.0 (we pin 10.4.0 — confirm no STRK20 fixes we need were added later).
- Xverse dapp-facing Wallet API status (Ready confirmed).
- Fee UX: wallet flows sponsor gas but not pool fees; shielded-token fee payment still being designed — don't promise a fee UX, re-check at build time.
- Mainnet discovery/indexer + proving endpoints for the SDK-direct dev path (Day-0 doc lists them as pending; wallet flows handle proving).
- Monorepo drift: `packages/sub_account_anonymizer` renamed/gone, new `packages/shadow_account_anonymizer` — irrelevant to this route, but noted.

## 11. Links

- Wallet API route: https://strk20-by-example.org/starknet-wallet-api/overview · starknet.js wiring: https://strk20-by-example.org/starknet-wallet-api/starknet-js
- Private DeFi through the Wallet API (open notes + invoke): https://strk20-by-example.org/starknet-wallet-api/private-defi
- Anonymizer anatomy / `privacy_invoke`: https://strk20-by-example.org/helpers/privacy-invoke · swap example: https://strk20-by-example.org/helpers/swap-helper · lending example: https://strk20-by-example.org/helpers/vesu-lending-helper
- SDK monorepo (reference packages + quickstart): https://github.com/starkware-libs/starknet-privacy
- Privacy pool (mainnet): https://voyager.online/contract/0x040337b1af3c663e86e333bab5a4b28da8d4652a15a69beee2b677776ffe812a
- WalletAccount guide (fetch before writing connect code): https://starknet-js.com/docs/next/guides/account/walletAccount/#with-get-starknet-v6
- Versions: `starknet@10.4.0` (npm `next`), get-starknet v6.0.3/6.0.4, `@starknet-io/types-js@0.10.3`, Ready extension
