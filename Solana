// jito.js
import axios from "axios";
import bs58 from "bs58";
import {
  SystemProgram,
  TransactionMessage,
  VersionedTransaction,
  PublicKey,
} from "@solana/web3.js";

const DEFAULT_ENDPOINTS = [
  // Prefer engines historically accepting lower tips first
  // Order can be overridden via JITO_ENDPOINTS env or constructor opts
  "https://tokyo.mainnet.block-engine.jito.wtf/api/v1/bundles",
  "https://amsterdam.mainnet.block-engine.jito.wtf/api/v1/bundles",
  "https://frankfurt.mainnet.block-engine.jito.wtf/api/v1/bundles",
  "https://ny.mainnet.block-engine.jito.wtf/api/v1/bundles",
  "https://mainnet.block-engine.jito.wtf/api/v1/bundles",
];

const DEFAULT_VALIDATORS = [
  "DfXygSm4jCyNCybVYYK6DwvWqjKee8pbDmJGcLWNDXjh",
  "ADuUkR4vqLUMWXxW9gh6D6L8pMSawimctcNZ5pGwDcEt",
  "3AVi9Tg9Uo68tJfuvoKvqKNWKkC5wPdSSdeBnizKZ6jT",
  "HFqU5x63VTqvQss8hp11i4wVV8bD44PvwucfZ2bU7gRe",
  "ADaUMid9yfUytqMBgopwjb2DTLSokTSzL1zt6iGPaS49",
  "Cw8CFyM9FkoMi7K7Crf6HNQqf4uEMzpKw6QNghXLvLkY",
  "DttWaMuVvTiduZRnguLF7jNxTgiMBZ1hyAumKUiL2KRL",
  "96gYZGLnJYVFmbjzopPSU6QiEV5fGqZNyN9nmNhvrZU5",
];

const JITO_JSONRPC_ID = 1;

function shuffle(arr) {
  const a = arr.slice();
  for (let i = a.length - 1; i > 0; i--) {
    const j = (Math.random() * (i + 1)) | 0;
    [a[i], a[j]] = [a[j], a[i]];
  }
  return a;
}

function sleep(ms) {
  return new Promise((r) => setTimeout(r, ms));
}

function parseRetryAfter(err) {
  const hdr =
    err?.response?.headers?.["retry-after"] ||
    err?.response?.headers?.["Retry-After"];
  if (!hdr) return null;
  // Header can be seconds or HTTP-date; we support seconds
  const asNum = Number(hdr);
  if (Number.isFinite(asNum) && asNum >= 0) return asNum * 1000;
  return null;
}

export class JitoJsonRpcClient {
  /**
   * @param {Connection} connection web3.js connection
   * @param {Keypair} payer fee payer / signer
   * @param {number} tipSol base SOL tip per bundle (e.g., 0.001)
   * @param {object} opts additional knobs
   */
  constructor(connection, payer, tipSol = 0.001, opts = {}) {
    this.connection = connection;
    this.payer = payer;
    this.tipLamportsBase = Math.floor((tipSol || 0.001) * 1_000_000_000);

    // Allow override via opts or JITO_ENDPOINTS env (comma-separated)
    const envList = (process?.env?.JITO_ENDPOINTS || "")
      .split(",")
      .map((s) => s.trim())
      .filter(Boolean);
    this.endpoints = Array.isArray(opts.endpoints) && opts.endpoints.length
      ? opts.endpoints
      : envList.length
      ? envList
      : DEFAULT_ENDPOINTS;
    this.validators = (opts.validators || DEFAULT_VALIDATORS).map(
      (v) => new PublicKey(v)
    );

    this.client = axios.create({
      headers: { "Content-Type": "application/json" },
      timeout: opts.timeoutMs || 15000,
    });

    // Endpoint health/cooldown state
    this.cooldowns = new Map(); // url -> ts(ms)
    this.endpointErrors = new Map(); // url -> count
    this.maxAttempts = opts.maxAttempts || 6;
    this.startTipMultiplier = opts.startTipMultiplier || 1.0;
    this.maxTipMultiplier = opts.maxTipMultiplier || 3.0;
    // Maintain a deterministic, cheapest-first order unless explicitly enabled
    this.shuffleCandidates = !!opts.shuffleCandidates; // default false
    this.confirmTimeoutMs = opts.confirmTimeoutMs || 90_000; // up to 90s
    this.pollIntervalMs = opts.pollIntervalMs || 2000;
  }

  getRandomValidator() {
    const idx = (Math.random() * this.validators.length) | 0;
    return this.validators[idx];
  }

  isCooledDown(url) {
    const until = this.cooldowns.get(url) || 0;
    return Date.now() < until;
  }

  markCooldown(url, ms) {
    const until = Date.now() + (ms || 10_000);
    this.cooldowns.set(url, until);
  }

  bumpError(url) {
    const n = (this.endpointErrors.get(url) || 0) + 1;
    this.endpointErrors.set(url, n);
    return n;
  }

  async buildTipTx(tipLamports) {
    const validator = this.getRandomValidator();
    console.log("🧩 Selected Jito Validator:", validator.toBase58());

    const { blockhash, lastValidBlockHeight } =
      await this.connection.getLatestBlockhash("confirmed");

    const tipMsg = new TransactionMessage({
      payerKey: this.payer.publicKey,
      recentBlockhash: blockhash,
      instructions: [
        SystemProgram.transfer({
          fromPubkey: this.payer.publicKey,
          toPubkey: validator,
          lamports: tipLamports,
        }),
      ],
    }).compileToV0Message();

    const tipTx = new VersionedTransaction(tipMsg);
    tipTx.sign([this.payer]);
    const tipSig = bs58.encode(tipTx.signatures[0]);
    // Encode tip transaction as base58 to match Jito sendBundle expectations
    const tipTxEncoded = bs58.encode(tipTx.serialize());

    return { tipTxEncoded, tipSig, blockhash, lastValidBlockHeight };
  }

  /**
   * Send a bundle across multiple endpoints with robust backoff & tip adaptation.
   * @param {string[]} encodedTransactions base58 transactions (without tip; we add it)
   * @returns {{success:boolean, bundleId?:string, tipSignature?:string, usedEndpoint?:string, signatures?:string[]}}
   */
  async sendBundle(encodedTransactions, tipLamportsOverride) {
    let tipMultiplier = this.startTipMultiplier;
    const endpoints = this.endpoints.slice();

    // Deterministic attempt order (cheapest-first). No random shuffling by default.
    let attempt = 0;
    let lastBundleId = null;
    let lastTipSig = null;
    let usedEndpoint = null;

    while (attempt < this.maxAttempts) {
      attempt += 1;
      // Filter cooled down endpoints; if all cooled, ignore cooldown this attempt
      const candidates = endpoints.filter((u) => !this.isCooledDown(u));
      // Keep declared order by default (cheapest-first). Optional shuffle via opts.
      const round = this.shuffleCandidates
        ? shuffle(candidates.length ? candidates : endpoints)
        : (candidates.length ? candidates : endpoints);

      const baseTipLamports = Number.isFinite(tipLamportsOverride)
        ? tipLamportsOverride
        : this.tipLamportsBase;
      const tipLamports = Math.min(
        Math.floor(baseTipLamports * tipMultiplier),
        Math.floor(baseTipLamports * this.maxTipMultiplier)
      );

      // Build tip tx for this attempt
      const { tipTxEncoded, tipSig } = await this.buildTipTx(tipLamports);
      // Transactions must be base58-encoded strings (tip first, then user txs)
      const bundle = [tipTxEncoded, ...(encodedTransactions || [])];

      // Try all endpoints in this attempt
      for (const url of round) {
        const payload = {
          jsonrpc: "2.0",
          id: JITO_JSONRPC_ID,
          method: "sendBundle",
          params: [bundle],
        };

        console.log("🚀 Broadcasting bundle:", {
          endpoint: url,
          tipLamports,
          attempt,
        });

        try {
          const res = await this.client.post(url, payload);
          const bundleId = res?.data?.result;
          if (bundleId) {
            console.log(`✅ Bundle accepted by ${url}`);
            lastBundleId = bundleId;
            lastTipSig = tipSig;
            usedEndpoint = url;
            return {
              success: true,
              bundleId,
              tipSignature: tipSig,
              usedEndpoint: url,
              signatures: [tipSig], // first is tip; the rest are swap txs (unknown until deserialized)
            };
          }
          // No result → treat as failure
          this.bumpError(url);
        } catch (err) {
          const status = err?.response?.status;
          const retryMs = parseRetryAfter(err);

          if (status === 429) {
            console.warn(`⏳ 429 from ${url}${retryMs ? ` (Retry-After ${retryMs}ms)` : ""}`);
            // backoff tip slightly & mark cooldown
            tipMultiplier = Math.min(tipMultiplier * 1.15, this.maxTipMultiplier);
            this.markCooldown(url, retryMs || 4_000);
          } else if (status >= 500 || status === 408) {
            console.warn(`⚠️ ${status} from ${url} — server busy; cooling endpoint`);
            this.markCooldown(url, 8_000);
          } else {
            console.warn(`⚠️ Send failed on ${url}:`, err?.message || status);
            this.bumpError(url);
            this.markCooldown(url, 4_000);
          }
        }
      }

      // If none succeeded in this round, exponential backoff w/ jitter
      const base = Math.min(15_000, 1000 * Math.pow(2, attempt));
      const jitter = Math.random() * 500;
      const delay = base + jitter;
      console.log(
        `⏳ All endpoints failed (attempt ${attempt}/${this.maxAttempts}). Backing off ${(delay / 1000).toFixed(
          2
        )}s... (tip x${tipMultiplier.toFixed(2)})`
      );
      await sleep(delay);
    }

    console.error("❌ Jito bundle failed to send after retries.");
    return { success: false, bundleId: lastBundleId, tipSignature: lastTipSig, usedEndpoint };
  }

  async getBundleStatuses(url, bundleIds) {
    const payload = {
      jsonrpc: "2.0",
      id: JITO_JSONRPC_ID,
      method: "getBundleStatuses",
      params: [bundleIds.map((b) => [b])],
    };
    try {
      const res = await this.client.post(url, payload);
      return res?.data?.result?.value || [];
    } catch (e) {
      return [];
    }
  }

  /**
   * Confirm by polling *both* Jito bundle status (if available) and RPC signatures.
   * We treat "confirmed"/"finalized" as success.
   * @param {{bundleId?:string, signatures?:string[], usedEndpoint?:string, timeoutMs?:number}} args
   */
  async confirm(args = {}) {
    const {
      bundleId,
      signatures = [],
      usedEndpoint = this.endpoints[0],
      timeoutMs = this.confirmTimeoutMs,
    } = args;

    const start = Date.now();

    while (Date.now() - start < timeoutMs) {
      // 1) Check Jito bundle status (fastest when available)
      if (bundleId && usedEndpoint) {
        const st = await this.getBundleStatuses(usedEndpoint, [bundleId]);
        const s0 = st?.[0];
        if (s0?.confirmation_status === "confirmed" || s0?.confirmation_status === "finalized") {
          console.log(`📊 Jito bundle confirmed (${s0.confirmation_status})`);
          return { confirmed: true, source: "jito", bundleStatus: s0 };
        }
      }

      // 2) Check RPC signature statuses
      if (signatures.length) {
        try {
          const resp = await this.connection.getSignatureStatuses(signatures, { searchTransactionHistory: true });
          const anyOk = (resp?.value || []).some((s) =>
            s && (s.confirmationStatus === "confirmed" || s.confirmationStatus === "finalized") && !s.err
          );
          if (anyOk) {
            console.log("📡 RPC shows transaction confirmed/finalized.");
            return { confirmed: true, source: "rpc" };
          }
        } catch (e) {
          // ignore transient errors
        }
      }

      await sleep(this.pollIntervalMs);
    }

    console.error(
      `❌ Transaction was not confirmed in ${(timeoutMs / 1000).toFixed(
        2
      )} seconds. It may still land later.`
    );
    return { confirmed: false };
  }
}
