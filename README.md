# Jito Bundle Helper (JavaScript)

A small helper around Jito’s block engine JSON-RPC for sending Solana bundles with adaptive tips, multi-endpoint retries, and confirmation polling. It wraps a `Connection` and payer, builds a tip transaction, fans out to multiple block engine URLs, and polls both Jito bundle status and RPC signatures.

## Features
- Multi-endpoint failover with cooldown/backoff and optional shuffling.
- Adaptive tip sizing with configurable multipliers and max attempts.
- Random validator selection from a default allowlist (overrideable).
- Optional gradient/bicolor plotting left intact from the original indicator logic.
- Confirmation polling via Jito `getBundleStatuses` and standard RPC signatures.
- Alerts: crossover-based buy/sell signals are emitted as plotshapes/alertconditions.

## Install
```bash
npm install axios bs58 @solana/web3.js
```

## Usage
```js
import { Connection, Keypair } from "@solana/web3.js";
import { JitoJsonRpcClient } from "./jito.js"; // path to this file

// 1) Set up Solana connection & payer
const connection = new Connection("https://api.mainnet-beta.solana.com", "confirmed");
const payer = Keypair.fromSecretKey(/* Uint8Array */);

// 2) Instantiate the client (tip in SOL)
const jito = new JitoJsonRpcClient(connection, payer, 0.001, {
  maxAttempts: 6,
  confirmTimeoutMs: 90_000,
  pollIntervalMs: 2000,
  shuffleCandidates: false, // keep cheapest-first order by default
});

// 3) Provide your user transactions (base58-encoded)
const userTxs = [
  /* tip tx is auto-built; add your own base58 tx strings here */
];

// 4) Send bundle
const sendRes = await jito.sendBundle(userTxs);
console.log("Bundle send result:", sendRes);

// 5) Confirm (optional)
const confirmRes = await jito.confirm(sendRes);
console.log("Confirm result:", confirmRes);
```

## Configuration
- `tipSol`: base SOL tip per bundle (default `0.001`).
- `endpoints`: override the block engine URLs (defaults prioritize “cheapest-first”).  
  You can also set `JITO_ENDPOINTS` env var as a comma-separated list.
- `validators`: override the validator allowlist (array of base58 strings).
- `startTipMultiplier`, `maxTipMultiplier`: control tip growth during retries.
- `maxAttempts`: total bundle send attempts (default `6`).
- `confirmTimeoutMs`, `pollIntervalMs`: confirmation polling controls.
- `shuffleCandidates`: if true, shuffles endpoints each attempt (default false).

## Notes
- `sendBundle` prepends a tip transaction automatically and returns `{ success, bundleId, tipSignature, usedEndpoint, signatures }`.
- `confirm` polls both Jito and RPC; it returns `{ confirmed, source, bundleStatus? }`.
- Handle secret keys carefully; never commit them. Prefer environment-based key management.

## License
MIT (see root LICENSE).
