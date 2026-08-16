# VeilPayouts 🎁

**Private business payouts on Starknet — pay anyone by link, they claim into a shielded balance.**

VeilPayouts is a privacy-native payouts rail built on [STRK20](https://strk20.starknet.io/). Companies pay people **without collecting wallet addresses and without exposing recipient balances**. The recipient gets a claim link; clicking it (once registered) lands the payout directly in their **shielded STRK20 balance** as an encrypted note — no public on-chain link between the payer, the amount, and the payee.

It solves the gap every STRK20 app hits: **you cannot privately transfer to someone who hasn't registered a viewing key yet.** VeilPayouts escrows the funds behind a secret, hands the secret off-chain as a claim link, and the recipient claims it into their own note the moment they're ready.

Built for the [STRK20 Private Sprint](https://strk20.starknet.io/hackathon) (Aug 14–31, 2026) — **IDEA-10, Business payouts API**.

---

## The flow

```
Company dashboard          Recipient
─────────────────          ─────────
1. Create payout ──► escrow STRK behind a secret (commitment on-chain)
2. Get claim link ──► share off-chain (email / chat / QR)
3.                   ◄── click link, connect wallet (register if new)
4.                   ◄── claim: prove the secret → payout lands as a
                         shielded note (no public leg)
5. Refund (optional) ──► unclaimed funds returned after expiry
```

- **Payer side**: shield STRK into the pool, then a private transfer funds the escrow. Public observers see a pool→escrow transfer, never *who* paid *whom* or *how much*.
- **Recipient side**: no wallet address needed up front — just a link. The claim is a private note; the recipient's balance stays shielded until they choose to unshield.
- **Business side**: a dashboard lists payouts (status: pending / claimed / expired) without ever revealing balances or the payer↔payee link on-chain.

## What is private, what is not

Precise hidden-vs-visible breakdown: see [docs/PRIVACY-MODEL.md](docs/PRIVACY-MODEL.md).

| Public | Private |
|---|---|
| The **shield deposit** (address, token, amount — STRK20 design) | The **payout amount** and **who funded it** (private transfer) |
| The **escrow commitment hash** (reveals nothing — Poseidon of the secret) | The **secret / claim link** (off-chain) |
| That a **claim happened** (a nullifier + open note) | **Who claimed** and **to whose balance it landed** |
| The recipient's **unshield**, if they choose to cash out | Which payout a withdrawal came from |

**Claim:** payout amounts, payer identity, payee identity, and the payer↔payee link are private. **Never claim:** the shield deposit or a final unshield are hidden.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Next.js frontend (two sides)                                │
│  • Business: create payouts, dashboard, refunds              │
│  • Recipient: claim links, shielded balance view             │
│  WalletAccountV6 via wallet-standard (Ready / Xverse)        │
└──────────────┬──────────────────────────────┬───────────────┘
               │ shield / unshield / transfer │ private transfer + invoke
               ▼                              ▼
┌──────────────────────────┐   ┌──────────────────────────────┐
│  STRK20 privacy pool     │   │  PayoutEscrow (Cairo helper) │
│  (mainnet 0x0403…812a)   │◄──┤  commitment → claim state    │
│  encrypted notes         │   │  machine, expiry, refunds    │
└──────────────────────────┘   └──────────────────────────────┘
```

- **Money layer:** the STRK20 pool — shield to fund, private transfer to escrow, claim as a private note, unshield to cash out.
- **Escrow layer:** `PayoutEscrow`, a stateful Cairo **anonymizer** in the documented `privacy_invoke` pattern — `Deposit` stores a commitment keyed by `poseidon(secret)`; `Claim` verifies the secret, marks it claimed, and returns an `OpenNoteDeposit` crediting the claimer's note. Adapted from the [STRK20-by-example escrow example](https://strk20-by-example.org/helpers/escrow), extended with expiry + refund.
- **Fairness:** commitments are domain-separated (`ESCROW_COMMITMENT_TAG:V1`); double-claims revert; funds are pullable by the pool only.

## Repository layout

| Path | What it is |
|---|---|
| `src/` | Next.js 16 frontend (business dashboard + recipient claim flow) |
| `cairo/` | `PayoutEscrow` — the stateful anonymizer contract |
| `docs/` | Architecture, privacy model, integration plan |
| `strk20.json` | Sprint submission — mainnet txs, contracts, demo |

## Getting started

```bash
npm install
cp .env.example .env.local    # add your Alchemy key
npm run dev                   # http://localhost:3000
```

Needs a privacy-enabled Starknet wallet — [Ready](https://www.ready.co/) works today.

## Sprint status

- [x] Idea locked (IDEA-10), repo rebranded, entry prepared
- [ ] STRK20 integration plan (via `strk20-privacy-integration` skill)
- [ ] Day 0: first mainnet pool transactions (shield / transfer / unshield)
- [ ] `PayoutEscrow` Cairo contract: commitments, claim, expiry, refund
- [ ] Frontend: create-payout flow, claim-link landing, dashboard
- [ ] Mainnet: 3 pool txs in `strk20.json`, live demo, demo video

## Links

- [STRK20 privacy pool](https://github.com/starkware-libs/starknet-privacy) · [STRK20 by example](https://strk20-by-example.org/) · [Escrow helper reference](https://strk20-by-example.org/helpers/escrow) · [Starter kit](https://github.com/Akashneelesh/strk20-starter-kit) (MIT, frontend base)
- Sprint: [Private Sprint](https://strk20.starknet.io/hackathon) · [IDEAS.md / IDEA-10](https://github.com/starkience/strk20-hackathon/blob/main/IDEAS.md)

## License

MIT. Frontend bootstrapped from the [STRK20 starter kit](https://github.com/Akashneelesh/strk20-starter-kit) (MIT).
