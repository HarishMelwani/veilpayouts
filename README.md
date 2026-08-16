# VeilPoker ♠️

**Provably fair Texas Hold'em on Starknet — chips you can't trace, cards nobody can see, a deal nobody can rig.**

VeilPoker is a privacy-native poker table built on [STRK20](https://strk20.starknet.io/), Starknet's privacy layer. It answers the oldest question in online poker — *"how do I know the house isn't cheating?"* — with cryptography instead of trust:

- **Chips are shielded.** You shield STRK into the privacy pool. Your buy-in, your stack, your winnings move as encrypted notes. Nobody can link your seat to your wallet.
- **Cards are private.** Hole cards are dealt under a **commit-reveal scheme** with a dealer seed that no single party controls. Even the table contract cannot see your hand before you reveal it.
- **The deal is provably fair.** The deck is derived from a seed that mixes every player's random commit, so cheating is *mathematically impossible*, not just disincentivized.

Built for the [STRK20 Private Sprint](https://strk20.starknet.io/hackathon) (Aug 14–31, 2026).

---

## What is private, what is not

We are precise about this — see [docs/PRIVACY-MODEL.md](docs/PRIVACY-MODEL.md) for the full threat model.

| Public | Private |
|---|---|
| Your **shielding deposit** (address, token, amount) | Note-to-note transfers: amounts and parties |
| Your **unshield withdrawal** (destination, amount) | Which deposit a withdrawal came from |
| Buy-in amount sent to a table escrow | That you are the one playing / your table stack |
| Pot size at showdown | Who folded, who called, your hole cards |
| Final hand outcome | Winner's link back to their public wallet |

**Claim:** identity privacy for players, card privacy until showdown, and provable fairness of the deal. **Never claim:** that deposits or withdrawals are hidden — shielding is not private, what you do afterwards is.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Next.js frontend (wallet connect, table UI, game client)    │
│  • WalletAccountV6 via wallet-standard (Ready / Xverse)      │
│  • STRK20 actions: deposit / transfer / withdraw / invoke    │
└──────────────┬──────────────────────────────┬───────────────┘
               │ shield / unshield            │ private tx + invoke
               ▼                              ▼
┌──────────────────────────┐   ┌──────────────────────────────┐
│  STRK20 privacy pool     │   │  VeilTable (Cairo contract)  │
│  (mainnet 0x0403…812a)   │◄──┤  seat registry, pot ledger,  │
│  encrypted notes,        │   │  commit-reveal dealing,      │
│  discovery + proving     │   │  payout via open notes       │
└──────────────────────────┘   └──────────────────────────────┘
```

- **Money layer:** the STRK20 pool. Shield to buy in, private-transfer to seat, winnings paid as private notes, unshield to cash out.
- **Game layer:** `VeilTable`, a Cairo contract (the "anonymizer" in STRK20 terms) that holds the pot as an open note and settles winners through the pool.
- **Fairness layer:** commit-reveal dealing. Every player commits a random nonce, the dealer seed is `hash(all commits)`, the deck is derived deterministically from it, and each player's cards are committed on-chain before the deal — verifiable by anyone after reveal.

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for the full design.

---

## Repository layout

| Path | What it is |
|---|---|
| `src/` | Next.js 16 frontend (wallet connect, table, game client) |
| `cairo/` | Cairo contracts — `VeilTable` (poker table / anonymizer) |
| `src/game/` | Poker engine: deck, shuffle, hand evaluation, seat logic |
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

- [x] Repository scaffolded, entry registered (registry PR)
- [ ] Day 0: first mainnet pool transactions (shield / transfer / unshield)
- [ ] STRK20 integration plan (via `strk20-privacy-integration` skill)
- [ ] `VeilTable` Cairo contract: seats, commit-reveal dealing, pot, payouts
- [ ] Game client: heads-up Hold'em table, betting rounds, hand evaluation
- [ ] Mainnet: 3 pool txs in `strk20.json`, live demo, demo video

## Links

- [STRK20 privacy pool](https://github.com/starkware-libs/starknet-privacy) · [STRK20 by example](https://strk20-by-example.org/) · [Starter kit](https://github.com/Akashneelesh/strk20-starter-kit) (MIT, forked as the frontend base)
- Sprint: [Private Sprint](https://strk20.starknet.io/hackathon) · [IDEAS.md / RFP-03](https://github.com/starkience/strk20-hackathon/blob/main/IDEAS.md)

## License

MIT. Frontend bootstrapped from the [STRK20 starter kit](https://github.com/Akashneelesh/strk20-starter-kit) (MIT).
