# VeilPoker Privacy Model

> Written to be precise, because overclaiming privacy is the fastest way to lose points on integration depth. This document states exactly what VeilPoker hides, what it does not, and what an observer can actually learn.

## Hidden vs visible, transaction by transaction

| Step | On-chain visibility | Notes |
|---|---|---|
| **1. Shield** (deposit STRK into pool) | PUBLIC: depositor address, token, amount | STRK20 design — deposits are screened by a compliance provider. Shielding is not private. |
| **2. Buy-in** (private transfer to table escrow) | PRIVATE: note-to-note, no amount, no parties | The transfer is encrypted; only the table's viewing key can open the note. |
| **3. Nonce commits + deal** | PUBLIC: commitments, community cards, deck seed | Fairness data must be public to be verifiable. Commitments reveal nothing about the nonces (Poseidon). |
| **4. Hole cards** | PUBLIC: commitment of each player's two cards | `poseidon(card, seat, seed)` — preimage is secret until reveal. |
| **5. Bets / pot** | PARTIAL: aggregate pot size public; per-player contributions private | The table contract records `(seat, amount)` in private state; the pot as an open note shows an amount but not who contributed what. |
| **6. Reveal** | PUBLIC: nonces, cards | Required for provable fairness. |
| **7. Settlement** | PRIVATE: winner receives a private note via the pool | Nobody can see which seat got the pot, and the seat itself is unlinkable to a wallet. |
| **8. Unshield** | PUBLIC: destination + amount | Which deposit it came from is hidden — a winner can unshield to a fresh wallet and the trail stops at the pool. |

## What an observer can actually learn

- **Identity:** shielded address, unshield destination, and (public) deposit amounts.
- **Table activity:** that a game is running, the pot size trajectory, deal timestamps.
- **Correlation risks (we warn users):** a distinctive buy-in amount shortly before a distinctive deposit is correlatable; timing of bets relative to public deposits can leak. Claim identity privacy, never claim amount privacy for the deposit/unshield steps.

## What is NOT claimed

- ❌ Deposit shielding is private — it is not (STRK20 design).
- ❌ Bet amounts are private — the pot aggregate is public.
- ❌ The winner is anonymous to the *other players at the table* — they see the winning cards and seat at showdown (that's poker).
- ❌ Cards are encrypted pool notes — they are protected by commit-reveal, not note encryption.

## What IS claimed

- ✅ **Identity privacy:** no on-chain observer can link a seat/wallet to a player's pool activity.
- ✅ **Card privacy:** until reveal, no party — including the table contract and the dealer — knows a player's hole cards.
- ✅ **Provable fairness:** the deck is derivable from the published seed and commitments; any attempt to rig the deal is publicly detectable.
- ✅ **Private payouts:** winnings arrive as encrypted notes; their link to a public wallet exists only if the winner chooses to unshield.

## Trust assumptions

1. **STRK20 pool** is honest (the sprint's whole premise).
2. **VeilTable contract** is verifiable open source; its dealing logic is deterministic and replayed client-side for verification.
3. **Client software** does not exfiltrate nonces/cards (standard assumption for any dapp; the on-chain trail is clean regardless).

## Future work (post-sprint)

- Stealth-account seats so even *which contract* receives the buy-in is unlinkable.
- Viewing-key proofs for selective disclosure (e.g., prove you hold a stack without revealing it).
