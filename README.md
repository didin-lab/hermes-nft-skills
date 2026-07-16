# Hermes NFT Mint Skills

A comprehensive collection of **Hermes Agent skills** for automating NFT minting operations on EVM chains — from single-wallet fast mints to 50+ wallet parallel broadcasts, stage-open sniping, contract security analysis, and whitelist campaign automation.

> Built for the Hermes Agent ecosystem. These are **agent skills** (SKILL.md files) that teach an LLM agent to execute NFT operations with precision, not standalone scripts.

---

## 🧩 Skills Overview

| # | Skill | Purpose | Key Capability |
|---|---|---|---|
| 1 | `nft-mint` | End-to-end NFT lifecycle | ERC-721/721A/SeaDrop, contract probing, mint discovery |
| 2 | `nft-fast-mint` | Single-wallet speed mint | Minimal RPC calls (4-6), ABI caching, pattern shortcuts |
| 3 | `nft-parallel-mint` | 50+ wallet batch mint | ThreadPoolExecutor, per-wallet nonces, gas-abort, pre-flight |
| 4 | `nft-stage-open-snipe` | Phase-open sniping | Pre-warm → tight-poll → fire at T0, sub-second broadcast |
| 5 | `nft-mint-preflight` | Multi-fleet health check | 50-wallet ETH scan + 8-account X token validation |
| 6 | `nft-toolkit` | NFT infrastructure | SIWE auth, calldata builder, GraphQL client, multi-RPC |
| 7 | `contract-analysis` | Pre-mint security | Bytecode scanning, selfdestruct/drain/fee-change detection |
| 8 | `nft-wl-social-campaign` | WL campaign automation | Reverse-engineer WL APIs, multi-account pipeline, spin/gacha |
| 9 | `nft-wl-submission` | WL form submission | Google Forms, Supabase, X-engagement gates, 50+ wallets |
| 10 | `cron-multi-rpc-fallback` | RPC resilience | Multi-RPC with self-healing, fallback chains, cron-safe |
| 11 | `evm-eip1559-fee-optimizer` | Gas optimization | EIP-1559 fee estimation, priority fee discovery, hard caps |

---

## 📊 Gas Rules

The user operates with a **strict gas ceiling**:

- **Hard cap**: 0.2–0.3 gwei max base fee. Anything above = **ABORT**.
- **Priority fee**: 0.01–0.05 gwei minimum for mempool inclusion.
- **Two-layer abort**: pre-flight (before any tx) + per-wallet (mid-batch spike).
- **Propagation**: abort flag must be explicit — never silently skip.

```python
# Pre-flight
if base_fee > w3.to_wei(max_gas_gwei, "gwei"):
    sys.exit("GAS ABORT: base_fee above cap")

# Per-wallet
if base_fee > max_gas_wei:
    return {"abort": True, "err": "GAS_ABORT"}
```

---

## 🔍 Contract Detection Flow

The `nft-mint` and `nft-fast-mint` skills share a unified contract probing pipeline:

### 1. Bytecode Analysis
```
eth_getCode → detect proxy pattern (EIP-1167: 45 bytes = minimal proxy)
```

### 2. ABI Resolution
```
Blockscout API (free) → extract ABI from implementation address
Fallback: 4byte.directory for selector lookup
Last resort: scan recent mint TXs for calldata
```

### 3. Mint Function Discovery (Priority Order)
```
1. freeMint() / freeMint(uint256)        — zero-cost
2. claim(uint256) / claim(address,uint256)
3. mint(uint256) / mint()                 — ERC-721A standard
4. mintPublic(address,address,address,uint256) — SeaDrop v2
5. quoteMint(address,uint256)             — Tessera free-then-paid
```

### 4. Price Detection
```python
# Try estimateGas with value=0
gas = contract.functions.mint(qty).estimate_gas({"from": addr, "value": 0})
# → price = 0 (free mint!)

# If revert: parse IncorrectPayment error (0x0d35e921)
# → extract actual price from error data
```

---

## 🚀 SeaDrop v2 Flow

The most common pattern for OpenSea drops:

```
Token Contract (EIP-1167 proxy)
  └─ Implementation (ERC-721SeaDrop)
       └─ SeaDrop v2 Router: 0x00005EA00Ac477B1030CE78506496e8C2dE24bf5
            └─ mintPublic(nftContract, feeRecipient, minter, qty)
```

**Calldata template** (bypass OpenSea API rate limits):
```
0x161ac21f                          ← selector
  + nftContract (32 bytes, left-padded)
  + feeRecipient (32 bytes)
  + minterIfNotPayer (32 bytes)     ← byte 80: swap per wallet
  + quantity (32 bytes, left-padded)
```

**Error signatures:**
- `0x13da22f2` — `NotActive(current, start, end)` — expected before drop
- `0x0d35e921` — `IncorrectPayment(expected, actual)` — price discovery

---

## ⚡ Parallel Mint Architecture

```
                    ┌──────────────┐
                    │  Pre-flight  │
                    │  ABI + Gas   │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
         ┌────▼────┐  ┌────▼────┐  ┌────▼────┐
         │ Wallet 1 │  │ Wallet 2 │  │ Wallet N │
         │ nonce=5  │  │ nonce=3  │  │ nonce=7  │
         │ sign+send│  │ sign+send│  │ sign+send│
         └────┬─────┘  └────┬─────┘  └────┬─────┘
              │            │            │
              └────────────┼────────────┘
                           │
                    ┌──────▼───────┐
                    │  Receipts    │
                    │  poll (240s) │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Report:     │
                    │  N/M success │
                    └──────────────┘
```

**Key constraints:**
- Max 20 concurrent workers (RPC rate limit)
- 10-20s delay before polling receipts (null = normal)
- Never share nonce state between parallel sends
- Auto-skip wallets with insufficient balance

---

## 🎯 Stage-Open Snipe (3-Phase)

The `nft-stage-open-snipe` skill implements a timing-critical pattern:

### Phase 1: Cron (T-3min)
```
Schedule cron 2-3 min BEFORE stage time, not at it.
Ensures script is loaded, RPC connected, ABI cached before T0.
```

### Phase 2: Pre-Warm (T-3min → T-5s)
```
1. connect_rpc() with fallback chain
2. Cache ABI + detect mint function
3. Detect price
4. Pre-fetch nonces (pending)
5. Pre-compute gas
6. Pre-build tx dicts
7. Balance pre-check
8. Gas abort check
```

### Phase 3: Fire (T0)
```
Tight-poll every 1.5s → stage_open → re-read base_fee → sign+send all
```

---

## 🛡️ Contract Security (Pre-Mint)

Before sending any transaction to an unfamiliar contract:

```
1. Bytecode scan → detect proxy, selfdestruct, delegatecall
2. Blockscout verification → is source verified?
3. Ownership check → renounced? multi-sig? EOA?
4. Drain patterns → sweepToken(), rescueETH(), withdrawAll()
5. Fee-change → can mint price change mid-drop?
6. Supply cap → totalSupply() vs maxSupply()
```

---

## 📝 Whitelist Campaign Pipeline

The `nft-wl-social-campaign` skill handles the full WL flow:

```
1. Reverse-engineer WL API
   ├─ /start → /step → /submit-tweet → /spin
   └─ Detect verification: real tweet required? API trust only?

2. Multi-account pipeline
   ├─ Social steps (API trust — no real X actions)
   ├─ Real tweet post (if backend verifies URL)
   ├─ Real follow (if user chooses "campur" mode)
   └─ Spin (5% win rate, 12h cooldown)

3. Daily cron retry
   ├─ Reuse existing tweets
   ├─ Idempotent: check cooldown before spin
   └─ Report per-account: WIN/LOSE/COOLDOWN
```

---

## 🔧 Wallet Infrastructure

| Fleet | Count | Source | Purpose |
|---|---|---|---|
| WCore | 1 | `~/wallets/wcore.key` | Operator wallet, funding source |
| w1–w50 | 50 | `wallets.json` | Hunt/mint wallets |
| w51–w57 | 7 | `wallets_pk.txt` (lines 51–57) | X account wallets |
| X accounts | 8 | `~/.x_creds_*` | Cookie-based X auth |

**Balance thresholds:**
- `> 0.00005 ETH` — funded
- `> 0.0001 ETH` — mint-ready (1 paid mint + gas)
- `> 0.0005 ETH` — 3-mint ready

---

## 📦 File Structure

```
~/.hermes/skills/web3/
├── nft-mint/          # Core NFT lifecycle
├── nft-fast-mint/     # Speed-optimized single mint
├── nft-parallel-mint/ # Multi-wallet broadcast
├── nft-stage-open-snipe/  # Timing-critical sniping
├── nft-mint-preflight/    # Fleet health checks
├── nft-toolkit/       # SIWE, calldata, GraphQL
├── contract-analysis/ # Security scanning
├── nft-wl-social-campaign/  # WL campaign automation
├── nft-wl-submission/ # WL form submission
├── cron-multi-rpc-fallback/ # RPC resilience
└── evm-eip1559-fee-optimizer/ # Gas optimization
```

---

## 🔗 Related Skills

| Skill | Purpose |
|---|---|
| `opensea-eligibility-checker` | SIWE + GraphQL drop eligibility |
| `opensea-collection-check` | Holdings, rarity, pre-reveal status |
| `seadrop-direct-mint` | Raw SeaDrop calldata mint (no API) |
| `x-multi-account` | Multi-account X/Twitter operations |
| `xurl` | Official X API v2 CLI |
| `web-wl-gate` | Browser-based WL gate solving |

---

## ⚠️ Common Pitfalls

1. **web3.py v7**: `signed.raw_transaction` (snake_case), NOT `rawTransaction`
2. **SeaDrop `transferFrom`**: use `safeTransferFrom` with `estimateGas × 1.3`
3. **Priority fee 0.02 gwei** → drops from mempool. Use ≥ 0.05 gwei.
4. **Stage drift**: re-query on-chain state at T0, never trust cron schedule
5. **Single RPC** = single point of failure. Always multi-RPC fallback.
6. **Blockscout 403**: add `User-Agent: Mozilla/5.0` header
7. **EIP-1167 proxy**: get bytecode first — 45 bytes = proxy, extract impl from bytes 10-29
8. **`totalSupply >= maxSupply`** → collection fully minted, HARD STOP
9. **Cron `no_agent=true`**: `sys.exit(1)` on 0/N success to trigger error alert
10. **OpenSea API 429**: rotate API key (`POST /auth/keys`) or use calldata template

---

## 🤖 Hermes Agent Integration

These skills are part of the **Hermes Agent** ecosystem. Each skill is a `SKILL.md` file that the agent loads on-demand:

```
User: "mint paralel 50 wallet ke contract ini"
Agent: loads nft-parallel-mint → detects contract → broadcasts → reports
```

Skills are loaded via `skill_view(name)` and executed in the agent's context. They contain:
- Full Python code patterns
- RPC call sequences
- Error handling recipes
- Pitfall documentation
- Reference implementations

---

## 📄 License

MIT — use, modify, and distribute freely.