# Mainnet Day 0 — VeilPayouts runbook

> Sprint eligibility needs **three mainnet transactions** that touched the STRK20 pool, each listed by hash in `strk20.json`. This is the checklist to get there. **Real money** — start with an amount you would not mind losing (a few STRK is plenty).

## Verified network values

| | Mainnet | Sepolia (dev) |
|---|---|---|
| Chain ID | `SN_MAIN` | `SN_SEPOLIA` |
| RPC | `https://rpc.starknet.lava.build` | `https://starknet-sepolia-rpc.publicnode.com` (free; see `src/utils/constants.ts`) |
| STRK20 pool | `0x040337b1af3c663e86e333bab5a4b28da8d4652a15a69beee2b677776ffe812a` | `0x0254a6b2997ef52e9f830ce1f543f6b29768295e8d17e2267d672c552cfe0d91` |
| Wallet | Ready extension, switched to Mainnet | Ready, Sepolia |

Pool on Voyager: https://voyager.online/contract/0x040337b1af3c663e86e333bab5a4b28da8d4652a15a69beee2b677776ffe812a

## What you need

- Ready wallet on **Mainnet**, funded with a few STRK (CEX withdrawal to Starknet, or bridged from Ethereum).
- Second Starknet address for the private-transfer leg (a second wallet, or a fresh account in the same wallet).

## The flow (three pool transactions)

1. **Register** — the wallet sets your viewing key on first STRK20 use (transparent; one time per address).
2. **Shield** — deposit a small amount of STRK into the pool. Emits `Deposit(user_addr, token, amount)`.
3. **Private transfer** — send a small amount to your second address. Note-to-note: no amount, no parties.
4. **Unshield** (optional but recommended) — withdraw back to a public address.

Do it through **VeilPayouts' own buttons** once Phase 2 ships (shield / transfer / unshield are the app's money primitives), or through the official app at https://strk20.starknet.io/app as a fallback.

## Gotchas that will confuse you

- **The tx sender is a relayer, not you.** Private transactions are submitted by rotating shared relayers — the sender address will be a stranger with a huge nonce, and your address appears nowhere in the calldata or signature. That's the system working. Eligibility is verified from the pool's own `Deposit` event, not the tx sender.
- **A shield is two steps.** The ERC-20 `approve` lands first, then the deposit. Two wallet prompts — that's expected, not a bug.
- **Notes mature ~10 blocks.** Freshly shielded funds aren't immediately spendable. Shield ahead of time; don't shield-then-transfer back-to-back.
- **Deposits are screened.** FPI screens the depositing address and signs every deposit; the pool verifies on-chain. A deposit can be declined by screening — that's a protocol outcome, not an app bug.
- **There is a pool fee per private operation.** Read `get_fee_amount` from the pool rather than assuming; it was ~4 STRK on mainnet at the time of writing. Budget for it and subtract it from any "max" amount.
- **Proving takes a while.** A privacy-pool tx verifies a STARK proof on-chain; `waitForTransaction` can take minutes. Keep the explorer link as the fallback.

## Privacy — be precise

| Public | Private |
|---|---|
| Deposit (address, token, amount) | In-pool transfers (amounts, parties) |
| Withdrawal (destination, amount) | Which deposit a withdrawal came from |
| Fact + timing of pool interaction | — |

Shield ahead of time, transfer later — the separation is what makes the flow unlinkable.

## After the three transactions

1. Copy the three tx hashes (from Voyager: https://voyager.online).
2. Paste them here — I'll verify each on-chain (exists, succeeded, touched the pool) and fill `strk20.json`:
   ```json
   { "transactions": ["0x…", "0x…", "0x…"], "contracts": [], "demo_video": "", "demo_url": "" }
   ```
3. Push. The hub updates within the half hour.
