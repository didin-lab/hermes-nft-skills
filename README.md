# Hermes NFT Mint Skills

A practical English guide to **Hermes Agent skills for minting NFTs on EVM chains** — from single-wallet fast mints to **50+ wallet parallel** broadcasts, **stage-open sniping**, contract security, eligibility checks, and whitelist campaigns.

> These are **agent skills** (`SKILL.md` playbooks). They teach an LLM agent *how* to mint safely and fast — not a one-click GUI.

**Repo:** https://github.com/didin-lab/hermes-nft-skills

---

## What this is

When a freemint / SeaDrop / allowlist phase opens, speed and discipline matter:

| Need | Skill |
|---|---|
| Understand contract + mint path | `nft-mint` / `nft-fast-mint` |
| Hit many wallets at once | `nft-parallel-mint` |
| Fire the second a stage opens | `nft-stage-open-snipe` |
| Check fleet ETH + X cookies | `nft-mint-preflight` |
| Who is on GTD/FCFS before open | `opensea-eligibility-checker` |
| Skip OpenSea API rate limits | `seadrop-direct-mint` / calldata template |
| Don’t get rugged | `contract-analysis` / `contract-scanner` |
| RPC doesn’t die mid-run | `cron-multi-rpc-fallback` |

---

## Skills overview

| # | Skill | Purpose |
|---|---|---|
| 1 | `nft-mint` | End-to-end lifecycle: probe → fund → mint → transfer |
| 2 | `nft-fast-mint` | Single-wallet speed (few RPC calls, ABI cache) |
| 3 | `nft-parallel-mint` | Concurrent mint across many wallets (per-wallet nonces) |
| 4 | `nft-stage-open-snipe` | Pre-warm → poll → fire at T0 |
| 5 | `nft-mint-preflight` | Fleet health: balances + account readiness |
| 6 | `nft-toolkit` | SIWE, calldata builder, GraphQL, multi-RPC helpers |
| 7 | `contract-analysis` | Pre-mint bytecode / drain / fee-change checks |
| 8 | `nft-wl-social-campaign` | Social-gated WL flows (API reverse + multi-account) |
| 9 | `nft-wl-submission` | Form/API allowlist submission at scale |
| 10 | `cron-multi-rpc-fallback` | Multi-RPC connect + self-heal for cron mints |
| 11 | `evm-eip1559-fee-optimizer` | EIP-1559 fee caps and priority fee discovery |
| 12 | `opensea-eligibility-checker` | SIWE + GraphQL signed-presale eligibility |
| 13 | `seadrop-direct-mint` | Direct SeaDrop calldata mint (bypass API) |

Related portfolio piece: [opensea-eligibility-check](https://github.com/didin-lab/opensea-eligibility-check).

---

## Features (what you get)

### 1) Contract intelligence first
- Detect EIP-1167 proxy vs implementation
- Resolve ABI (explorer / 4byte / recent mint txs)
- Discover mint function order: `freeMint` → `claim` → `mint` → SeaDrop `mintPublic` → `quoteMint`

### 2) SeaDrop-aware path
OpenSea drops often **mint on SeaDrop, not the token contract**:

```text
Token (proxy) → ERC721SeaDrop impl
              → SeaDrop v2 router mintPublic(nft, feeRecipient, minter, qty)
```

- Prefer signed-presale eligibility via **SIWE + GraphQL** before open
- Live stage probe: `build_mint_tx` → tx payload vs **422** (not listed) vs **409** (not started)
- Hard 429 on mint-builder → switch to **calldata template** + direct broadcast

### 3) Parallel multi-wallet mint
- One wallet = one nonce stream (no shared-nonce races)
- Funding WCore → wallets is **serial**; mints wallet → contract are **parallel**
- Auto-skip empty wallets; gas abort mid-batch

### 4) Stage-open snipe (3 phases)
1. **Cron early** (T−2–3 min) — load script, connect RPC  
2. **Pre-warm** — ABI, price, nonces, gas, pre-built tx dicts  
3. **Fire** — tight-poll until stage open → sign+send all

### 5) Strict gas discipline
Typical operator ceiling (example ops setup):

- Hard **base fee cap** (abort if above user max)
- Two-layer abort: **pre-flight** + **per-wallet** mid-run
- Never silently submit at higher gas when user said “more = cancel”

### 6) RPC resilience
- Primary private RPC + public fallbacks
- `connect_rpc()` / `ensure_w3()` self-heal mid-cron
- Mandatory for timed snipes

---

## Why use Hermes skills for minting

| Approach | Pros | Cons |
|---|---|---|
| Manual MetaMask | Simple | Slow, one wallet, easy to miss T0 |
| Random bot scripts | Fast-looking | Broken nonces, no gas abort, rug blind |
| **Hermes skill stack** | Probe + eligibility + parallel + snipe + RPC heal | Needs agent/ops setup |

Benefits:

- **Coverage** — many wallets in one open window  
- **Latency** — pre-warm so T0 isn’t cold-start  
- **Safety** — paid mint ask-first, gas abort, contract scan  
- **Honesty** — report real tx hashes; never invent success  

---

## When to use which skill

| Situation | Load |
|---|---|
| “Mint 1 freemint this contract” | `nft-fast-mint` |
| “Mint all funded wallets” | `nft-parallel-mint` |
| “Auto mint the second FCFS opens” | `nft-stage-open-snipe` + multi-RPC |
| “Who is on GTD?” | `opensea-eligibility-checker` |
| “OpenSea mint API 429 forever” | `seadrop-direct-mint` / template |
| “Is this contract shady?” | `contract-analysis` |
| “Are wallets funded / X cookies alive?” | `nft-mint-preflight` |

---

## Mint flow (high level)

```text
1. get_drop / SIWE eligibility   → who & when
2. contract probe + security     → how & is it safe
3. preflight balances            → who can pay gas/price
4. pre-warm (ABI, nonces, gas)   → ready before T0
5. fire (parallel or snipe)      → broadcast
6. poll receipts                 → report N/M + hashes
```

### SeaDrop calldata template (concept)

When API mint-builder is unusable:

```text
selector mintPublic
  + nftContract (32b)
  + feeRecipient (32b)
  + minterIfNotPayer (32b)  ← swap per wallet
  + quantity (32b)
```

Broadcast to SeaDrop router; do not call `mintSeaDrop` on the token (reverts `OnlyAllowedSeaDrop`).

---

## Wallet fleets (typical layout)

| Fleet | Count | Source | Role |
|---|---|---|---|
| WCore | 1 | `~/wallets/wcore.key` | Operator / funding |
| Hunt wallets | w1–w50 | `wallets_pk.txt` / `wallets.json` | Parallel mint |
| X-linked | w51–w57 + wcore | identity map | WL social + mint |

**Always derive address from PK** before operating. Auto-skip insufficient balance.

---

## Gas rules (operator standard)

```python
# Pre-flight
if base_fee > max_gas_gwei:
    abort("GAS ABORT")

# Per-wallet mid-batch
if base_fee > max_gas_gwei:
    return {"abort": True, "err": "GAS_ABORT"}
```

- Abort flag must be **explicit** (never silent skip)
- Priority fee too low → mempool drop (tune with fee optimizer skill)

---

## Pitfalls

1. **Paid mint** — always confirm with user if `value > 0`  
2. **Public ≠ WL** — public eligibility is not a competitive win  
3. **`build_mint_tx` 409** — stage not started; not “not listed”  
4. **`build_mint_tx` 422** — not on *active* allowlist  
5. **Stage schedule drift** — re-query at fire time  
6. **Shared nonce** — never parallelize one sender’s funding txs  
7. **Single RPC** — single point of failure on snipes  
8. **EIP-1167** — 45-byte code = proxy; mint via SeaDrop / impl path  
9. **web3.py v7** — `signed.raw_transaction` (snake_case)  
10. **Never invent** tx hashes / “success” without receipt  

---

## Compare: mint styles

| Style | Wallets | Latency | Best for |
|---|---|---|---|
| Fast mint | 1 | Medium | Probe + single claim |
| Parallel | N | Medium | Freemint coverage |
| Stage snipe | N | Lowest at T0 | Timed FCFS/public open |
| Manual | 1 | High | Learning only |

---

## FAQ

**Q: Free mint failed with IncorrectPayment — is it free?**  
A: Simulate with `value=0`. If only paid path works, stop and ask before sending.

**Q: OpenSea says minting but GraphQL all false?**  
A: You’re not on signed stages; wait for public or skip.

**Q: Cron at stage time still late?**  
A: Fire cron **early**, pre-warm, then poll for open — don’t cold-start at T0.

**Q: Can I mint without OpenSea API key?**  
A: Yes for many SeaDrop paths (direct calldata / on-chain). Eligibility GraphQL uses SIWE session, not always a REST key.

**Q: How many wallets concurrent?**  
A: Cap by RPC rate limits (often ~20 workers); more wallets = waves.

---

## Security notes

- Private keys: `chmod 600`, never commit  
- Don’t paste PKs in chat logs  
- Scan unknown contracts before first send  
- Respect chain ToS / project rules; automation at your own risk  

---

## Companion (Indonesian, shorter)

A simpler Indonesian version lives on Notion (portfolio hub):

→ https://app.notion.com/p/Skill-Mint-NFT-di-Hermes-ringkas-3a6dddfb96cd81008895ce80b5518ee7

---

## Related repos

- [opensea-eligibility-check](https://github.com/didin-lab/opensea-eligibility-check) — SIWE + GraphQL WL checks  
- [hermes-x-multi-account](https://github.com/didin-lab/hermes-x-multi-account) — multi-account X ops for social WL  

---

## License

MIT (documentation). Third-party chains, OpenSea, and project contracts have their own terms.
