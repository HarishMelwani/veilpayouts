# Bug report — STRK20 shield fails with UNKNOWN_ERROR on Sepolia (Ready wallet)

**Date:** 2026-08-16 · **Status:** to be reported to the STRK20 team (sprint Telegram group / issue tracker)

## Symptom

Every STRK20 shield (deposit) attempt on **Sepolia** fails immediately with:

```
Action failed
An error occurred (UNKNOWN_ERROR)
```

No transaction is submitted. The error comes from the wallet (Wallet API error object), not from the chain or the dapp.

## Reproduction — fails identically on three independent apps

| App | Result |
|---|---|
| VeilPayouts (our app, https://github.com/HarishMelwani/veilpayouts) | UNKNOWN_ERROR |
| Official starter demo (starknet-privacy-starter.vercel.app) | UNKNOWN_ERROR |
| Official STRK20 app (strk20.starknet.io/app) | UNKNOWN_ERROR |

Because the official app and the starter demo (same code as ours) fail identically, the bug is not in any dapp's code.

## Environment

- Wallet: **Ready** (extension), network **Sepolia**, STRK balance **100** (faucet).
- Capability check passes: `supportedWalletApi` reports ≥ 0.10.3 (STRK20-capable).
- Chain: SN_SEPOLIA (chainId `0x534e5f5345504f4c4941`) — confirmed via RPC.
- Pool on Sepolia: `0x0254a6b2997ef52e9f830ce1f543f6b29768295e8d17e2267d672c552cfe0d91` — confirmed live (class resolves).
- STRK token: `0x04718f5a0fc34cc1af16a1cdee98ffb20c31f5cd61d6ab07201858f4287c938d` — confirmed deployed on Sepolia.

## Suspected cause

The wallet's STRK20 preparation for **Sepolia** — proving service and/or deposit-screening (FPI) endpoint — failing before a transaction is built. The same wallet flow is widely reported working on **mainnet** (many sprint teams have shipped mainnet STRK20 transactions), which points at the testnet-side backend rather than the wallet client.

## Asked to try

- [ ] Update the Ready extension (chrome://extensions → Developer mode → Update) and retry.
- [ ] Confirm whether any other participant sees this on Sepolia (sprint Telegram group).

## Impact on VeilPayouts

Low for the sprint: eligibility requires **mainnet** transactions only; Sepolia was a dev rehearsal. Product build continues; end-to-end verification will happen on mainnet once mainnet STRK is available.
