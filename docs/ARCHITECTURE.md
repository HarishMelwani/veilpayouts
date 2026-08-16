# VeilPoker Architecture

> Draft v0 — will be revised by the STRK20 integration plan (see `STRK20_INTEGRATION_PLAN.md` once written). Scope for the sprint: **heads-up (2-player) Texas Hold'em**, architected to scale to a full ring table.

## Goals

1. **Provable fairness** — no party (dealer, contract, other players) can influence or see the cards before they should.
2. **Identity privacy** — a player's seat cannot be linked to their public wallet by on-chain observers.
3. **Shielded money** — buy-ins, stacks, and payouts move through the STRK20 pool as encrypted notes.
4. **Ship on mainnet** — three real STRK20 pool transactions, a live demo, and a 3-minute video by Aug 31.

## Components

### 1. Frontend (Next.js 16, TypeScript)

- Forked from the [STRK20 starter kit](https://github.com/Akashneelesh/strk20-starter-kit): wallet picker (get-starknet v6, `eip1193Adapters: []`), `WalletAccountV6` connection, `strk20InvokeTransaction` action plumbing, `strk20Balances` reads.
- `src/game/` — pure TS poker engine: deck model, Fisher-Yates shuffle seeded from the deal seed, hand evaluator (5/7-card), seat/round state machine. Unit-testable without a chain.
- `src/app/` — table UI: seat panel, hole cards, community cards, betting controls (fold / check / call / raise), pot display, receipts.

### 2. STRK20 pool (money layer)

- **Shield** (`deposit`): public by design — address, token, amount on-chain.
- **Private transfer** (`transfer`): encrypted note-to-note; no amount, no parties.
- **Unshield** (`withdraw`): public destination + amount; which deposit it came from is hidden.
- **Invoke** (`invoke` via `privacy_invoke`): anonymous contract calls — the pool sends funds to a contract (our `VeilTable`), the contract runs logic atomically, and returns `OpenNoteDeposit`s to refill notes.

### 3. VeilTable (Cairo contract — the anonymizer)

The poker table itself, deployed as a Starknet contract that the pool can `privacy_invoke` against. Responsibilities:

- **Seat registry**: `seat -> stealth public key`, `seat -> shielded stack (note)`. Joining = private transfer of the buy-in to the table's escrow.
- **Commit-reveal dealing**:
  - `commit(seat, nonce_commitment)` — each player commits `hash(nonce)` before the deal.
  - `deal()` — dealer seed = `hash(all nonce_commitments)`; deck derived deterministically; hole cards committed on-chain; community cards posted.
  - `reveal(seat, nonce, cards)` — verified against the commitment; the deck derivation is replayed publicly so anyone can verify no card was swapped.
- **Pot ledger**: blinds and bets tracked as `(seat, amount)` — the aggregate is public (pot size), individual contributions are private.
- **Settlement**: at showdown, the pot (held as an open note) is split to winners via `OpenNoteDeposit`s; the winning hand is provably the best given the revealed cards.

> Honest scope note: the bet/raise game loop lives client-side with the table contract as the settlement + fairness anchor. A full on-chain betting state machine is a stretch goal; we will not fake it in the README if it doesn't land.

## Fairness scheme (commit-reveal)

1. Every player generates a random `nonce` and submits `commitment = poseidon(nonce)` before the deck is created.
2. The table's deal transaction consumes all commitments and computes `seed = poseidon(commitments...)`.
3. The deck is a deterministic permutation of 52 cards from `seed` (Fisher-Yates with a CSPRNG derived from `seed`).
4. Hole cards for each seat are committed as `poseidon(card, seat, seed)` and published.
5. At reveal, anyone can recompute the deck from `seed` and verify: (a) the published commitments match the revealed cards, (b) the cards came from the canonical deck order, (c) no card appears twice.

No single party controls `seed` — a cheating dealer would need to control every player's commitment. This is the "cheating is mathematically impossible" property.

## Data flow — a hand

```
1. Players shield STRK (pool deposit) — public.
2. Players buy in: private transfer to VeilTable escrow — private amount/parties.
3. Dealer (table contract) starts hand: players commit nonces.
4. deal(): seed -> deck -> hole card commitments + community cards posted.
5. Betting rounds (client-driven, pot recorded in table contract state).
6. Showdown: reveal nonces + cards; verify commitments + deck derivation.
7. Settlement: table contract emits OpenNoteDeposits to winners; pot refilled via pool.
8. Cash out: unshield (public withdrawal).
```

## Mainnet targets

| Value | Address |
|---|---|
| STRK20 pool | `0x040337b1af3c663e86e333bab5a4b28da8d4652a15a69beee2b677776ffe812a` |
| Chain | `SN_MAIN` |
| RPC | `https://rpc.starknet.lava.build` (public) / Alchemy |
| VeilTable (to deploy) | TBD — recorded in `strk20.json` |

## Non-goals (this sprint)

- Multi-table tournaments, rake, side pots, all-in insurance.
- Fully on-chain betting state machine (see scope note above).
- Card privacy via encrypted STRK20 notes *for the cards themselves* — cards are protected by commit-reveal, not by the pool's note encryption. The pool protects the *money*.
