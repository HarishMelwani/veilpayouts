# VeilPayouts Privacy Model

> Precise, because overclaiming privacy is the fastest way to lose points on integration depth. This states exactly what VeilPayouts hides, what it does not, and what an observer can learn.

## Hidden vs visible, step by step

| Step | On-chain visibility | Notes |
|---|---|---|
| **1. Creator shields STRK** | PUBLIC: depositor address, token, amount | STRK20 design — deposits are screened (FPI) and public. Done ahead of time, unlinked to any payout. |
| **2. Fund escrow** (private transfer → escrow) | PRIVATE: note-to-note, no amount, no parties | The payout amount and the payer's identity never hit a public leg. |
| **3. Escrow Deposit** (invoke) | PUBLIC: the commitment hash only | `poseidon(ESCROW_COMMITMENT_TAG, secret)` — preimage is secret; the hash reveals nothing. |
| **4. Claim link shared** | OFF-CHAIN | The secret travels by link (email/chat/QR). Any bearer can claim — by design. |
| **5. Recipient claims** (open note + invoke) | PARTIAL: an open-note amount + a nullifier | The open-note amount is plaintext by STRK20 design (measured at execution); **who** claimed it and **whose balance** it landed in are hidden. |
| **6. Recipient unshields** | PUBLIC: destination + amount | Which payout funded the withdrawal is hidden — the trail stops at the pool. |

## What an observer can actually learn

- That **a payout product is in use** (deposits, escrow commitments, claims).
- The **aggregate claim amount** of each escrow interaction (open-note plaintext).
- **Timing** of deposits vs claims.
- **Correlation risks (we warn users):** a distinctive shield amount shortly before a distinctive claim is correlatable; shield ahead of time and claim later. Claim identity privacy, never amount privacy for open notes or the public legs.

## What is NOT claimed

- ❌ The shield deposit is private — it is not (STRK20 design, screened by FPI).
- ❌ The claim amount is private — open notes are plaintext by design.
- ❌ The final unshield is private — withdrawals are public legs.
- ❌ The claim link is tamper-proof — it's a bearer secret; treat it like cash. A leaked link lets anyone claim (the payout goes to the first claimant).

## What IS claimed

- ✅ **Payout amounts are private** — the escrow funding is a private transfer; no public leg names the amount.
- ✅ **Payer identity is private** — observers see pool → escrow, never who initiated it.
- ✅ **Payee identity is private** — the claim credits a shielded note; only the claimant's viewing key can open it.
- ✅ **The payer↔payee link is private** — nothing on-chain connects the creator to the claimer.
- ✅ **Recipients need no registration to be paid** — the escrow + claim-link pattern solves STRK20's "must be registered to receive" gap.

## Trust assumptions

1. **STRK20 pool** is honest (the sprint's premise).
2. **PayoutEscrow** is verifiable open source; only the pool (`privacy_contract`) can drive it — nobody can withdraw escrowed funds directly.
3. **Claim links** are transported securely by the app (HTTPS); the secret is never stored in the app's backend.
4. **Client software** does not exfiltrate secrets (standard dapp assumption; the on-chain trail is clean regardless).

## Future work (post-sprint)

- Expiry + auto-refund as first-class escrow operations (sprint stretch goal).
- Viewing-key proofs for selective disclosure ("prove you were paid without revealing how much").
- Payouts to email/phone identifiers with onboarding that doesn't require understanding wallets.
