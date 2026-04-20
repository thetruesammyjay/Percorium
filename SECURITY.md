# Security Policy

## Supported Versions

Percorium is currently in the **Colosseum Frontier Hackathon 2026** build phase and deployed on **Solana Devnet only**. Mainnet has not launched.

| Version / Phase       | Supported          |
|-----------------------|--------------------|
| Devnet (Hackathon)    | ✅ Active          |
| Mainnet (Post-Audit)  | 🔜 Not yet live    |

> **Note:** During the hackathon phase, no real user funds are at risk — all activity is on Solana Devnet. Bug reports are still welcome and taken seriously; they inform the pre-mainnet audit and bug bounty programme.

---

## Reporting a Vulnerability

**Do NOT open a public GitHub issue for security vulnerabilities.**

If you discover a security issue, please disclose it responsibly via one of the following channels:

- **Primary:** Open a [GitHub Security Advisory](https://github.com/thetruesammyjay/Percorium/security/advisories/new) (private, encrypted disclosure)
- **Backup:** Email with `[PERCORIUM SECURITY]` in the subject line via the contact listed on the GitHub profile

### What to Include

Please provide as much of the following as possible:

1. **Description** — A clear summary of the vulnerability
2. **Impact** — What an attacker could achieve (fund loss, manipulation, DoS, privacy breach, etc.)
3. **Steps to Reproduce** — Minimal reproduction steps or a proof-of-concept (PoC)
4. **Affected Component** — Which program instruction, account, or module is impacted
5. **Suggested Fix** — If you have one (optional but appreciated)

### Response Timeline

| Stage                           | Target Time     |
|---------------------------------|-----------------|
| Acknowledgement of report       | Within 48 hours |
| Initial triage & severity assessment | Within 5 business days |
| Patch / mitigation deployed     | Depends on severity (see below) |
| Public disclosure (coordinated) | After patch is live |

---

## Severity Classification

Percorium uses a four-tier severity model aligned with the CVSS framework, adapted for on-chain DeFi protocols:

### 🔴 Critical
Direct, exploitable loss of user funds or protocol solvency. Examples:
- Draining the Global House Vault or any Slab PDA without authorization
- Minting synthetic ETF tokens without depositing Palm USD collateral
- Bypassing the 50% OI cap to over-lever the protocol
- Oracle manipulation leading to artificial NAV inflation/deflation that extracts collateral
- Any reentrancy attack that results in double-spend or balance corruption

**Response:** Devnet pause + immediate patch. Post-mainnet: protocol freeze, coordinated fix, full post-mortem.

### 🟠 High
Significant financial risk or privilege escalation with non-trivial preconditions. Examples:
- Bypassing the 24-hour LP withdrawal cooldown without exploiting it for oracle manipulation
- Draining the Insurance Fund PDA without triggering the intended ADL flow
- CPI signer seed forgery allowing unauthorized instruction execution (Jupiter, Kamino, Encrypt CPIs)
- Circuit breaker bypass that keeps a slab live during oracle disagreement

**Response:** Patch within 72 hours.

### 🟡 Medium
Disruption to protocol operation or user privacy without direct fund loss. Examples:
- Griefing attacks that cause Jito bundles to fail repeatedly for targeted users
- Leaking FHE-encrypted position metadata through side-channel emissions in transaction logs
- Compute unit exhaustion causing circuit breaker to fail silently
- Revenue distribution logic producing incorrect 65/20/15 splits due to rounding errors

**Response:** Patch within 14 days.

### 🟢 Low / Informational
Minor issues, best-practice deviations, or theoretical attack vectors with negligible real-world impact. Examples:
- Minor discrepancies in NAV calculation math with no exploitable consequence
- Missing input validation on non-critical instruction parameters
- Doc/comment inconsistencies that could mislead future contributors

**Response:** Addressed in next scheduled release.

---

## In-Scope

The following components are in scope for vulnerability reports:

### On-Chain Programs (`programs/percoria/`)
- All Anchor instruction handlers (vault, slab, ETF, perp, oracle, revenue)
- All PDA account definitions and constraint validations
- Math library functions (NAV, uPnL, funding rate, A/K scaling, leverage throttle)
- CPI bindings (Jupiter, Kamino, Encrypt)
- Custom error codes and their enforcement boundaries

### Key Attack Surfaces
| Component | Concern |
|---|---|
| `mint_etf.rs` | Minting without full hedge, OI cap bypass |
| `open_perp.rs` | Leverage limit bypass, margin calculation |
| `liquidate.rs` | Incorrect liquidation trigger, griefing |
| `withdraw_lp.rs` | 24h cooldown bypass, flash-drain |
| `update_nav.rs` | Oracle result tampering, TEE attestation bypass |
| `distribute_revenue.rs` | Incorrect split logic, overflow/underflow |
| `fund_insurance.rs` | Unauthorized drain of Insurance Fund PDA |
| `pause_slab.rs` | unauthorized pause/unpause, circuit breaker bypass |
| All CPI handlers | Signer seed forgery, account substitution |

### Frontend (`apps/web/`)
- Wallet connection security (Phantom, Backpack, Solflare)
- Bundle simulation logic in `packages/jito-client/`
- Private key / seed phrase exposure risks

---

## Out of Scope

The following are **not** in scope and will be closed without action:

- Vulnerabilities in external protocols Percorium integrates with (Jupiter, Kamino, Switchboard, Jito Block Engine, Encrypt Protocol, Palm USD, xStocks.fi) — report these to their respective teams
- Issues on Solana mainnet (Percorium is not live on mainnet)
- Theoretical issues with no realistic attack vector on devnet
- Findings related to centralization risks that are acknowledged design decisions (e.g., team-gated slab creation during hackathon phase — see CONTEXT.md)
- UI/UX bugs that do not involve security
- Issues in third-party npm/cargo packages not directly modified by Percorium

---

## Security Architecture Overview

Percorium implements multiple layers of defence-in-depth:

### On-Chain Guardrails

| Mechanism | Description |
|---|---|
| **Reentrancy Protection** | All state-modifying instructions protected via Anchor account constraints; no cross-instruction state corruption |
| **CPI Signer Checks** | All CPIs (Jupiter, Kamino, Encrypt) verify the PDA signing authority matches expected seeds before executing |
| **50% OI Hard Cap** | Perpetual open interest is hard-capped at 50% of Spot TVL per slab — enforced at the program level |
| **24-Hour LP Cooldown** | Withdrawal lock is a program constraint (`LpDepositAccount.unlock_ts`), not a UI gate — cannot be bypassed client-side |
| **Isolated Risk Slabs** | Each index lives in its own PDA — a bug or Oracle manipulation on one Slab cannot drain liquidity from another |
| **Circuit Breaker** | If >2 of 3 oracle sources disagree, the Slab is auto-paused on-chain. NAV updates are rejected until consensus is restored |
| **Insurance Fund** | 20% of protocol treasury revenue is diverted to a cold-storage PDA backstop. Separate authority from operations |

### Oracle Security

The NAV oracle is the highest-priority attack surface for any synthetic asset protocol:

- **Switchboard V3 TEE Functions** — Oracle computation runs inside a hardware-attested Trusted Execution Environment. The Percorium team cannot modify the TEE code post-deployment.
- **Multi-Source Aggregation** — Pyth, Chainlink, and DEX TWAP (Raydium/Orca) are aggregated inside the TEE
- **Outlier Discarding** — Any feed deviating >2% from the median is automatically excluded
- **Deviation Circuit Breaker** — Oracle disagreements trigger an automatic slab pause before any NAV update is applied
- **Atomic Same-Block Hedging** — Even if a NAV snapshot is slightly stale, the physical hedge executes at the same market price in the same block — no exploitable gap between oracle reading and hedge execution

### FHE Privacy Guarantees

- **User Encrypted State** — All `UserPosition` and `PerpPosition` balances, trade history, and unrealised PnL are Encrypt Protocol FHE ciphertexts stored on-chain
- **No Plaintext Emissions** — Position data is never emitted in program logs or events in plaintext form
- **User-Only Decryption** — Only the user's own private key can decrypt their position; the Percorium program cannot
- **Public by Design** — Slab configuration (weights, debt ceiling), aggregate OI, and Global Vault TVL remain public for protocol health transparency

### Atomicity Guarantee

All mint, hedge, and encryption operations execute as a single Jito Bundle:

```
TX 1: mint_etf (Percorium)
TX 2: physical hedge via Jupiter CPI (weighted basket purchase)
TX 3: encrypt_position (Encrypt FHE)
```

If **any** transaction in the bundle fails, the **entire bundle reverts atomically**. There is no exploitable intermediate state where a user holds synthetic tokens without a corresponding physical hedge.

---

## Pre-Mainnet Audit Plan

Before any mainnet deployment, Percorium will:

1. **Formal Protocol Audit** — Engage a reputable Solana/Anchor-specialised audit firm (e.g., OtterSec, Sec3, Neodyme, or Trail of Bits)
2. **Math Audit** — Formal verification of NAV, funding rate, A/K scaling, and leverage throttle formulas
3. **Public Bug Bounty Programme** — Open a time-boxed bug bounty on Immunefi or equivalent platform prior to mainnet
4. **Staged Rollout** — Mainnet launch with TVL caps that are raised incrementally as the protocol proves stability

---

## Acknowledgements

We are grateful to the security researchers, white-hat hackers, and DeFi community members who responsibly disclose vulnerabilities. Contributors who discover and responsibly report Critical or High severity issues will be acknowledged publicly (with their permission) and considered for recognition in the mainnet bug bounty programme.

---

*Last updated: April 2026 — Colosseum Frontier Hackathon build phase*
