# CONTEXT.md — Percorium Builder Context

> **Purpose:** This file is the canonical context document for any AI assistant, contributor, or LLM helping build Percorium. Read this before writing a single line of code. It contains the full mental model, design rationale, constraint boundaries, and decision history behind every architectural choice.

---

## 1. What Percorium Is

Percorium is a **sovereign on-chain layer for synthetic indices** built on Solana. It is not a perp DEX. It is not a lending protocol. It is not a copy of Synthetix. It is the **missing primitive** that sits between tokenised equities (xStocks.fi) and leveraged perpetual markets (Drift, Phoenix) — enabling users to compose, mint, and trade private custom index funds with atomic on-chain hedging.

**One-line pitch:** *Create your own S&P 500 or custom ETF on-chain, trade leverage on it, and do all of it privately — no KYC, no front-running, no position leakage.*

### The Name

- **Percolator** = Solana's internal transaction scheduler (Toly's project)
- **Emporium** = Market
- **Percorium** = A market built on Solana's percolation engine

---

## 2. Hackathon Context

**Event:** Colosseum Frontier Hackathon 2026
**Platform:** Solana Playground (beta.solpg.io) for initial build; monorepo for full development
**Target Tracks (Superteam Bounties):**
1. **Encrypt Protocol (FHE)** — Build applications using Encrypt's Fully Homomorphic Encryption on Solana
2. **Palm USD** — Build DeFi primitives using Palm USD as the native stablecoin

**Why these tracks are perfect fits:**
- Encrypt FHE is live on Solana devnet as of March 31, 2026 — Percorium uses it for all user position encryption (balances, trade history, PnL)
- Palm USD is a real, institutionally-backed stablecoin (Sharia-compliant, AED/SAR-backed) — it is Percorium's unit of account and collateral layer

**Deployment target:** Solana devnet for hackathon. Mainnet post-audit.

---

## 3. Core Problem Statements

Every design decision in Percorium traces back to solving one of these three problems:

### Problem 1: No Privacy-Preserving Index Exposure
Retail users who want diversified exposure to the S&P 500 or Nasdaq 100 cannot access it on-chain without:
- KYC'ing to a centralised platform (which routinely suffers data breaches)
- Leaking their portfolio composition on a public blockchain (enabling front-running and copy-trading against their positions)

**Percorium's solution:** xStocks.fi provides the compliant tokenised equity layer (no user KYC needed at the protocol level). Encrypt FHE encrypts all user balances and trade history at the L1 state level.

### Problem 2: Delta-Neutral Solvency Without a Central Counterparty
Synthetic asset protocols (Synthetix, early Mirror) failed because their global debt pool meant one manipulated asset could cascade across the entire protocol.

**Percorium's solution:** The Hub-and-Slab model. Each synthetic index lives in its own isolated Slab PDA. Risk is mathematically sandboxed per index. The Global House Vault backstops all slabs but cannot be drained by a single slab failure.

### Problem 3: MEV and Position Leakage for Whale Traders
Large positions on public blockchains are visible. Sandwich attacks and copy-trading erode alpha for serious traders.

**Percorium's solution:** Encrypt FHE encrypts all on-chain state at the position level. MEV bots cannot read what they cannot decrypt. Jito bundle atomicity ensures mint/hedge happens in a single block with no exploitable intermediate state.

---

## 4. Architecture Decisions and Rationale

### Decision 1: Hub-and-Slab vs Pure Global Pool

**Chosen:** Hub-and-Slab (Global Vault + Isolated Slab PDAs per index)

**Why not a single global pool?**
- Drift's April 2026 hack ($285M, North Korean social engineering) demonstrated the catastrophic blast radius of a single global pool
- A manipulated oracle on one index in a global pool affects all LPs
- Isolated Slabs contain each index's debt ceiling and OI cap independently

**Why keep a Global Vault at all?**
- Pure isolated pools murder capital efficiency (each slab needs its own liquidity)
- The Global House Vault is the backstop counterparty for ALL perp trades across all slabs
- LPs deposit once to the vault and earn fees from all protocol activity
- Kamino V2 yield stacking on idle capital is only possible at the global level

**The trade-off accepted:** Isolated slabs reduce capital efficiency vs a single pool. This is a deliberate choice — safety over efficiency at this stage.

---

### Decision 2: Jito Bundle Atomicity for Hedging

**Chosen:** All mints and long positions are hedged in the same block via Jito bundles (all-or-nothing)

**Why atomic hedging?**
- The protocol must be delta-neutral at all times
- If a mint succeeds but the hedge fails, the protocol is directionally exposed to market movement
- A failed hedge in a volatile market can make the protocol insolvent

**The trade-off accepted:** Bundle failures (rare but possible) mean the user's transaction simply doesn't go through and they retry. Higher Jito tip costs vs non-bundled protocols.

**CU Optimisation Strategy:**
1. Pre-simulate every bundle with exact accounts + recent blockhash
2. Set `computeUnitLimit = Math.ceil(actual * 1.15)` — no blind 1.4M requests
3. Pack high-value instructions first (tip/CU auction math)
4. Use v0 transactions + LUTs aggressively to shrink account lists
5. Split slabs with >1.2M CU into 2-3 txs within the same bundle (Jito supports up to 5)

---

### Decision 3: Switchboard V3 TEEs for Oracles

**Chosen:** Switchboard V3 Functions running inside Trusted Execution Environments (TEEs)

**Why TEEs for oracle computation?**
- TEE-enforced code cannot be tampered with post-deployment — the hardware attests to the exact code running
- Multi-source aggregation (Pyth + Chainlink + DEX TWAP) with outlier discarding happens inside the TEE
- Even the Percorium team cannot manipulate the oracle logic post-deployment

**Oracle gaming mitigations:**
- Strict deviation threshold: feeds >2% from median are discarded
- Circuit breaker: if >2 of 3 sources disagree, the slab auto-pauses
- Same-block hedging: even if a NAV snapshot is slightly stale, the hedge fills at market price simultaneously — no exploitable gap
- Isolated slabs: a gamed oracle on one slab has zero effect on other slabs

**Why not just Pyth?**
- Single source = single point of failure
- Multi-source TEE aggregation is the gold standard

---

### Decision 4: Encrypt FHE for User Privacy

**Chosen:** Encrypt Protocol (Fully Homomorphic Encryption) on Solana

**What is encrypted:**
- User token balances (Palm USD, synthetic ETF tokens)
- All trade history
- Open perp positions (size, entry NAV, unrealized PnL)

**What is NOT encrypted (public by design):**
- Slab configurations (weights, asset list, debt ceiling) — transparency for trust
- Global Vault TVL — LPs need to verify the protocol's health
- Aggregate OI per slab — needed for funding rate calculation

**Why FHE and not MPC (Arcium/Umbra)?**
- FHE operates on encrypted state at the L1 level — the encrypted values live in on-chain accounts and computation happens without decryption
- MPC (Arcium) requires off-chain computation nodes; FHE is fully on-chain
- Umbra/Arcium are better for private token transfers; Encrypt FHE is better for encrypted DeFi state (balances, positions, history)
- Encrypt FHE just dropped on Solana devnet (March 31, 2026) — perfect timing for the hackathon

**Trade-off accepted:** FHE computation is more expensive in CU than plaintext operations. Users will pay slightly higher fees for privacy — acceptable for the target user (whale, institutional, privacy-conscious retail).

---

### Decision 5: Socialized ADL (Not Targeted ADL)

**Chosen:** Drift's socialized ADL model (proportional haircuts across all profitable positions)

**Why not Hyperliquid's model (target the most profitable)?**
- Hyperliquid's ADL targets the largest winners — penalising the best traders
- Drift's socialized model distributes haircuts proportionally — everyone with a profitable position takes a small haircut rather than one person taking all the pain
- Community preference was clearly for Drift's model when both were active (Q1 2026)
- Toly (Anatoly Yakovenko) implemented socialized ADL in Percolator for this reason

**When does A/K scaling trigger?**
- Only when total payouts exceed the Slab's debt ceiling
- The scaling factor = `DebtCeiling / TotalPayout`
- This is a last-resort safety net — the 50% OI cap is designed to prevent this from ever triggering under normal conditions

---

### Decision 6: 24-Hour LP Withdrawal Cooldown

**Chosen:** 24-hour withdrawal cooldown on LP deposits to the Global House Vault

**Why?**
- Flash-drain attacks: without a cooldown, an attacker could deposit LP liquidity, manipulate an oracle to extract funds, then withdraw — all in one block
- Drift's attack required Circle to freeze USDC on Twitter because there was no protocol-level withdrawal lock — Percorium makes this a program constraint, not an emergency social media appeal
- Protects the insurance fund from being emptied before it can backstop losses

**Trade-off accepted:** LPs have 24-hour illiquidity. This is disclosed upfront. The yield (Kamino base APY + 15% protocol fees) must justify this lock period. Most institutional LPs accept 24h locks as standard.

---

### Decision 7: xStocks.fi for Equity Exposure

**Chosen:** xStocks.fi (SPYx, QQQx, AAPLx, NVDAx) as the underlying tokenised equity layer

**Why xStocks.fi?**
- 1:1 physically backed SPL tokens (custodied via Backed.fi)
- Fully composable on Solana DeFi
- Users are NOT buying stock ownership — they receive price exposure only (dividends reinvested or paid in stables)
- Team-level KYC with xStocks; users do not need individual KYC for price exposure derivatives

**Regulatory positioning:**
- Percorium offers synthetic price exposure, not securities ownership
- No voting rights, no dividends, no custody — purely delta exposure
- xStocks handles the RWA compliance backend
- Non-US focused deployment (no CFTC/SEC targeting)
- This is the same legal framework as any Solana perp protocol

---

## 5. What Percorium Is NOT

Understanding what NOT to build is as important as knowing what to build:

| Percorium is NOT | Why |
|---|---|
| A copy of Synthetix | Synthetix used a global debt pool — Percorium uses isolated slabs |
| A fork of Drift | Drift is a pure perp DEX; Percorium builds on physical spot ETFs first |
| A privacy mixer/tumbler | FHE encrypts DeFi state, not token transfers (use Umbra for that) |
| A stock brokerage | No custody, no ownership, no KYC — synthetic price exposure only |
| A yield aggregator | Kamino yield is a baseline feature, not the core product |
| A token launchpad | The protocol token comes post-hackathon to bootstrap liquidity |

---

## 6. Product Decisions: UX Abstraction

The technical complexity of Percorium must be **completely invisible** to end users. Here is the mapping from technical terminology to user-facing language:

| Technical Term | User-Facing Label |
|---|---|
| Skew-Based Funding Rate | "Expected Daily Yield (%)" |
| A/K Socialized Haircut | "Protocol Risk Balancer" |
| Slab PDA | "Index Vault" |
| Global House Vault | "Liquidity Pool" |
| Jito Bundle | (invisible — just fast) |
| FHE Encrypted Position | "Private Portfolio" |
| Compute Unit Limit | (invisible — handled automatically) |
| Dynamic Leverage Throttle | "Risk Meter" (slider UI) |
| 50% OI Cap | "Max Open Positions" |
| Debt Ceiling | "Index Capacity" |

**The elevator pitch for users:**
> "Create your own on-chain ETF or trade 5x leverage on the S&P 500. Everything is private. No KYC. Bring any token, we swap it for you."

---

## 7. The Monorepo Structure Rationale

Percorium uses a **pnpm + Turborepo monorepo** for the following reasons:

1. **Shared types** — The Anchor IDL types are generated once and consumed by both the web frontend and the test suite from a single source of truth (`packages/sdk`)
2. **Shared Jito client** — Bundle construction logic is complex enough to warrant its own tested package, reused across the frontend and scripts
3. **Independent deployability** — The web app can be deployed to Vercel independently from the Anchor program deployment
4. **Colosseum submission** — Judges can navigate a clean monorepo far more easily than a flat repository

---

## 8. Token Economic Context

**Hackathon phase (current):** No protocol token. Revenue flows to treasury PDA.

**Post-hackathon (planned):**
- Launch protocol governance token
- Token bootstraps initial liquidity for the Global House Vault
- Token-holders vote on: K constants per slab, debt ceilings, OI caps, revenue split adjustments
- Fully permissionless Slab creation gated by token deposit (spam prevention)

**Revenue Split (fixed at protocol level, not governance-adjustable during hackathon):**
- 65% → Protocol Treasury (PDA)
- 20% → Index Creator Royalty (creator's wallet)
- 15% → LP Rewards (proportional to xPalmUSD share)
- 20% of Treasury → Cold Storage Insurance Fund PDA

---

## 9. Key PDAs and Seeds

All Program Derived Addresses use deterministic seeds for reproducibility:

```rust
// Global House Vault
seeds = [b"global_vault"]

// LP Deposit (per user)
seeds = [b"lp_deposit", user_pubkey.as_ref()]

// Insurance Fund
seeds = [b"insurance_fund"]

// Slab (per index)
seeds = [b"slab", slab_id.as_ref()]  // slab_id = creator-defined UTF-8 string

// User Position (per user, per slab)
seeds = [b"user_position", user_pubkey.as_ref(), slab_id.as_ref()]

// Perp Position (per user, per slab)
seeds = [b"perp_position", user_pubkey.as_ref(), slab_id.as_ref()]

// Oracle Feed (per slab)
seeds = [b"oracle_feed", slab_id.as_ref()]
```

---

## 10. Integration Contract: External Protocols

### Palm USD
- Used as: Primary collateral, unit of account, LP deposit token
- Integration: CPI token transfers (SPL Token standard)
- Devnet mint: Use official Palm USD devnet mint address (check docs.palmusd.com)
- No custom logic needed — treat as any SPL token

### Encrypt Protocol (FHE)
- Used as: Encryption of user balances and trade history
- Integration: Encrypt Solana bindings (Rust crate)
- All UserPosition account fields that contain balance/history/PnL MUST be FHE ciphertext types
- Decryption is user-side only (user's private key)
- Bounty alignment: document all FHE usage in `docs/colosseum/BOUNTY_ALIGNMENT.md`

### xStocks.fi
- Used as: Underlying assets for equity index Slabs
- Integration: Standard SPL token accounts for SPYx, QQQx, AAPLx, NVDAx
- The Slab holds physical xStocks tokens as the 1:1 backing for synthetic ETF mints
- Percorium does NOT modify xStocks tokens — it holds and releases them

### Jupiter (CPI)
- Used as: Executing physical hedges when users mint or open long perps
- Integration: Jupiter CPI (Cross-Program Invocation)
- All swaps are bundled with the mint/perp instruction via Jito
- Must pass exact account metas and slippage parameters

### Switchboard V3
- Used as: NAV oracle for all Slabs
- Integration: Switchboard V3 Functions (TEE-based off-chain computation with on-chain result verification)
- The TEE Function code must be published and verifiable
- Guardrail logic (multi-source, outlier discard, circuit breaker) must be hardcoded in the Function, not the Anchor program

### Kamino V2
- Used as: Yield stacking for idle LP capital in the Global House Vault
- Integration: Kamino CPI (deposit/withdraw from Kamino vaults)
- Only idle capital (not currently backing active perps) earns yield
- Yield is harvested periodically and added to LP fee distribution

### Jito Bundles
- Used as: Atomic multi-transaction execution (mint + hedge + encrypt = one block)
- Integration: Jito Block Engine API (TypeScript client in `packages/jito-client`)
- The Rust program does NOT know about bundles — bundle ordering is handled client-side
- Pre-simulate ALL bundles before submitting (exact accounts, 15% CU margin)

---

## 11. Known Limitations and Accepted Constraints

| Constraint | Status | Mitigation |
|---|---|---|
| FHE is computationally expensive (higher CU) | Accepted | Jito bundle CU optimisation absorbs most overhead |
| Bundle failures (all-or-nothing) cause user retries | Accepted | Pre-simulation reduces failure rate to <5% in testing |
| 24h LP cooldown is user-hostile | Accepted | Yield APY justifies lock; disclosed upfront |
| xStocks.fi token availability on devnet | Unknown | May need mock SPL tokens for devnet testing |
| Encrypt FHE devnet instability (new protocol) | High risk | Build FHE integration as an optional wrapper; core logic works without it |
| LUT limits for 10-asset slabs | Managed | Setup script creates LUTs before user interaction |
| Oracle TEE setup complexity | High | Switchboard dashboard + example Function code in `docs/ORACLE_GUIDE.md` |

---

## 12. What to Build First (Hackathon Priority Order)

For the Colosseum submission, build in this exact order:

**Week 1:**
1. `initialize_vault` — Global House Vault PDA with Palm USD
2. `deposit_lp` + `withdraw_lp` — With 24h cooldown enforcement
3. `create_slab` — One slab for SP500-Index (SPYx + QQQx + SOL + BTC)
4. `mint_etf` + `redeem_etf` — With Jupiter CPI hedge (mock Jupiter on devnet if needed)

**Week 2:**
5. Switchboard V3 TEE oracle integration — NAV feed for the slab
6. `open_perp` + `close_perp` — Basic 5x leverage with uPnL calculation
7. `settle_funding` — Skew-based funding rate settlement

**Week 3:**
8. Encrypt FHE integration — Encrypt UserPosition accounts
9. `distribute_revenue` — 65/20/15 revenue split
10. Frontend: Landing, Index Browse, Mint/Redeem panel, Jupiter Swap widget

**Week 4:**
11. Jito bundle integration in frontend
12. Full test suite (all 12 test files)
13. Colosseum submission docs + demo video

**Post-hackathon (Coming Soon):**
- Token-Gated Conviction Forums
- Isolated Lending (60% LTV)
- Delta-Neutral Yield Vaults
- Permissionless Slab Creation
- Mainnet migration

---

## 13. Success Criteria

**Minimum viable demo (must have):**
- [ ] Global House Vault deployed on devnet
- [ ] At least 1 Slab (SP500-Index) live with real NAV
- [ ] Mint + redeem synthetic ETF token working
- [ ] At least basic perp (open/close) working
- [ ] Encrypt FHE encrypting user positions (even if just balances)
- [ ] Palm USD as collateral throughout
- [ ] Jupiter swap widget in frontend
- [ ] Demo video showing the full flow

**Strong submission (nice to have):**
- [ ] 2 live slabs (SP500-Index + Nasdaq100-5x)
- [ ] Full funding rate settlement
- [ ] LP deposit/withdraw with yield
- [ ] Revenue distribution working
- [ ] Full frontend with normie-friendly UX copy
- [ ] All 12 test files passing

**Bounty unlock criteria:**
- Encrypt track: FHE-encrypted user positions + trade history on-chain ✅
- Palm USD track: Palm USD as primary collateral + unit of account throughout ✅

---

## 14. Questions and Design Debates (Resolved)

**Q: Should Slab creation be permissionless or team-gated?**
> For hackathon: team-gated (prevents spam on devnet demo). Post-launch: fully permissionless with governance token deposit as spam deterrent.

**Q: Should we use USDT instead of Palm USD for wider liquidity?**
> No. Palm USD is a Superteam bounty track. Using it is a direct unlock. Palm USD also has real institutional backing (Sharia-compliant, AED/SAR) which strengthens the "institutional-grade" narrative.

**Q: Should we use Umbra/Arcium instead of Encrypt for privacy?**
> Umbra/Arcium are better for private token *transfers*. Encrypt FHE is better for encrypted DeFi *state* (balances, positions, history). Percorium's privacy requirement is encrypted state, not private transfers. Use Encrypt. Also it's the bounty track.

**Q: What happens if a Jito bundle fails?**
> The entire bundle reverts atomically. The user retries. No partial state. This is by design — the delta-neutral invariant is more important than UX convenience. Pre-simulation reduces failure rate to <5%.

**Q: Do users need KYC?**
> No. xStocks.fi handles the RWA compliance layer. Users receive synthetic price exposure, not stock ownership. Percorium is a DeFi protocol, same legal category as any Solana perp DEX.

---

*Last updated: April 2026 — Colosseum Frontier Hackathon build phase*