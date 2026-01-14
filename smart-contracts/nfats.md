# Non-Fungible Allocation Token Standard (NFATS) — Business Requirements

**Status:** Draft
**Last Updated:** 2025-01-13

---

## Executive Summary

The Non-Fungible Allocation Token Standard defines a system for bespoke capital deployment deals between depositors and Halos. Unlike LCTS (which pools users into shared generations), NFATS treats each deal as an individual, non-fungible position represented by an NFAT (Non-Fungible Allocation Token).

Depositors queue sUSDS individually. The Halo claims from queues when deals are struck, minting an NFAT that represents the capital deployment. All deal terms, reward schedules, and maturity conditions are tracked offchain, giving maximum flexibility for bespoke arrangements.

**Key principle:** Onchain = minimal custody and ownership tracking. Offchain = all deal terms and lifecycle management.

---

## Why NFATS Exists

NFATS solves a different problem than LCTS:

| Scenario | Best Fit |
|----------|----------|
| Many users, same terms, shared capacity | LCTS |
| Individual deals, bespoke terms, named counterparties | NFATS |

### When to Use NFATS

- Asset manager partnerships with negotiated terms
- Deals where each depositor has different yield, duration, or conditions
- Situations requiring transferable positions (secondary market, collateralization)
- Regulated contexts where counterparty identity matters

### When to Use LCTS

- Open participation with uniform terms
- Capacity-constrained strategies where fair distribution matters
- Scenarios where fungibility and pooling are desirable

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         DEAL QUEUE                               │
│                                                                  │
│   Depositor A: [10M sUSDS]    ←── deposit / withdraw anytime    │
│   Depositor B: [5M sUSDS]                                        │
│   Depositor C: [0]                                               │
│                                                                  │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  │  Halo claims from queue
                                  │  (when deal is struck)
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                         DEAL NFAT                                │
│                                                                  │
│   NFAT #1: 10M principal, Depositor A, minted 2025-01-15        │
│   NFAT #2: 3M principal, Depositor B, minted 2025-01-20         │
│   NFAT #3: 2M principal, Depositor B, minted 2025-01-22         │
│                                                                  │
│   Each NFAT = one bespoke deal                                   │
│   Terms tracked offchain in Halo Artifact                        │
│                                                                  │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  │  Halo distributes rewards /
                                  │  redeems at maturity
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                      NFAT HOLDER                                 │
│                                                                  │
│   Receives rewards during lifecycle                              │
│   Receives principal + final reward at maturity                  │
│   Can transfer/sell NFAT anytime                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Components

### 1. Deal Queue

Each depositor has their own individual queue — not shared, not pooled.

**Depositor actions:**
- `deposit(amount)` — Add sUSDS to your queue
- `withdraw(amount)` — Remove sUSDS from your queue (anytime before Halo claims)

**Halo actions:**
- `claim(depositor, amount)` — Take sUSDS from a depositor's queue, mint NFAT

**Key behaviors:**
- No queue cap — depositors can queue unlimited sUSDS
- Halo can claim any amount up to the queued balance
- Partial claims create one NFAT; depositor retains remaining queue balance
- Depositor can withdraw anytime before claim (queue is non-binding until claimed)

### 2. Deal NFAT (ERC-721)

Minted when Halo claims from a queue. Each NFAT represents one bespoke deal.

**Onchain data (minimal):**

| Field | Description |
|-------|-------------|
| `tokenId` | Unique identifier |
| `principal` | Original sUSDS amount claimed |
| `depositor` | Original depositor address |
| `mintedAt` | Timestamp when deal was struck |

**Offchain data (Halo Artifact / deal database):**
- Deal terms (yield rate, duration, special conditions)
- Reward schedule (periodic, at maturity, claimable, etc.)
- Underlying deployment details
- Maturity conditions and redemption terms

**Transferability:**
- NFATs are transferable by default
- Halo can optionally restrict transfers to whitelisted addresses (for KYC/regulatory requirements)

### 3. Redemption and Rewards

Multiple patterns supported — orchestrated offchain, executed onchain by the Halo:

**Pattern A: Maturity Redemption**
- Halo redeems a deal by specifying the NFAT and amount
- NFAT is burned
- sUSDS (principal + reward) transferred to current NFAT holder

**Pattern B: Periodic Rewards**
- Halo distributes reward to a specific NFAT
- Reward transferred to current NFAT holder
- NFAT remains active

**Pattern C: Claimable Rewards**
- Halo sets a claimable amount for a specific NFAT
- NFAT holder can claim at any time
- Claimed amount transferred to holder, claimable balance resets

**Pattern D: Partial Redemption**
- Halo partially redeems an NFAT (returns some principal + reward)
- NFAT's recorded principal is reduced
- Assets transferred to holder
- NFAT remains active with reduced principal

The Halo chooses which pattern(s) to use per deal based on offchain terms.

---

## Deal Lifecycle

**1. Deposit**
- Depositor deposits sUSDS into their individual queue
- Queue balance increases

**2. Negotiation (offchain)**
- Halo and Depositor negotiate deal terms (yield, duration, reward schedule, etc.)
- Terms recorded in Halo Artifact

**3. Claim (deal struck)**
- Halo claims from depositor's queue (specifying amount)
- sUSDS transferred to Halo's ALMProxy
- NFAT minted to depositor
- Queue balance reduced by claimed amount

**4. Deployment**
- Halo deploys capital to underlying strategy (RWA, structured credit, custodian, etc.)

**5. Lifecycle**
- Rewards distributed per offchain terms (using any of the reward patterns)
- NFAT can be transferred/sold at any time
- New holder receives future rewards

**6. Maturity**
- Halo redeems the deal (principal + final reward)
- NFAT burned
- sUSDS transferred to current holder

---

## Behaviors

### Deal Queue Behaviors

**Depositor actions:**
- **Deposit**: Depositor adds sUSDS to their queue; increases their queued balance
- **Withdraw**: Depositor removes sUSDS from their queue (only possible before Halo claims)
- **View balance**: Anyone can check the queued balance for any depositor

### Deal NFAT Behaviors

**Halo actions:**
- **Claim**: Halo takes sUSDS from a depositor's queue and mints an NFAT to that depositor
  - Specifies: which depositor, how much to claim
  - Results in: sUSDS moves to Halo, new NFAT minted with unique ID
- **Redeem**: Halo closes out a deal at maturity
  - Specifies: which NFAT, how much sUSDS to return
  - Results in: NFAT burned, sUSDS transferred to current holder
- **Distribute reward**: Halo sends reward to an NFAT holder
  - Specifies: which NFAT, reward amount
  - Results in: sUSDS transferred to current holder, NFAT remains active
- **Partial redeem**: Halo returns some principal + reward while keeping deal active
  - Specifies: which NFAT, principal reduction, reward amount
  - Results in: NFAT's principal reduced, assets transferred to holder
- **Set claimable**: Halo sets an amount the NFAT holder can claim
  - Specifies: which NFAT, claimable amount
  - Results in: claimable balance updated for that NFAT

**Holder actions:**
- **Claim rewards**: NFAT holder pulls any claimable balance
  - Results in: claimable sUSDS transferred to holder, balance resets to zero
- **Transfer**: NFAT holder transfers the NFAT to another address (standard ERC-721)
  - New holder inherits all rights (future rewards, redemption)

**View functions:**
- Get principal amount for an NFAT
- Get original depositor for an NFAT
- Get mint timestamp for an NFAT
- Get claimable balance for an NFAT

### Optional: Transfer Restrictions

- Halo can enable a whitelist mode where only approved addresses can receive NFAT transfers
- Halo can add/remove addresses from the whitelist
- When whitelist is active, transfers to non-whitelisted addresses are blocked

---

## NFATS vs LCTS Comparison

| Aspect | LCTS | NFATS |
|--------|------|------|
| **Model** | Pool / ETF | Individual deals |
| **Position type** | Fungible shares | Non-fungible NFAT |
| **Terms** | Same for all in generation | Bespoke per deal |
| **Queue** | Shared across generation | Individual per depositor |
| **Capacity allocation** | Proportional distribution | Per-deal (Halo decides) |
| **Settlement** | Batch (weekly cycle) | Per-deal (anytime) |
| **Transferability** | Non-transferable shares | Transferable NFAT |
| **Exit before settlement** | Withdraw from active generation | Withdraw from queue before claim |
| **Reward mechanism** | rewardPerToken accumulator | Flexible (offchain-driven) |
| **Onchain complexity** | Higher (generations, settlement) | Lower (queue + NFAT) |
| **Offchain complexity** | Lower (uniform terms) | Higher (per-deal tracking) |

---

## Integration Notes

### Halo Sentinel

TBD — Sentinel integration for automated claims, reward distribution, and redemptions.

### ALM Controller Compatibility

NFATS requires custom ALM controller integration (similar to LCTS). The Halo's ALMProxy holds claimed sUSDS and sources redemption/reward payments.

### Halo Artifact

Deal terms should be recorded in the Halo Artifact for transparency and auditability. The NFAT's `tokenId` serves as the key linking onchain position to offchain terms.

---

## Design Rationale

### Why Individual Queues?

Shared queues (like LCTS generations) make sense when all participants receive identical treatment. For bespoke deals, individual queues:
- Allow depositors to signal interest without commitment
- Let the Halo selectively accept deals
- Avoid complexity of proportional distribution
- Enable partial claims (multiple deals with same depositor)

### Why NFATs?

ERC-20 tokens imply fungibility — any token is interchangeable with any other. When deals have different terms, forcing them into a fungible token creates friction:
- Transfer restrictions feel like hacks
- Per-holder accounting becomes complex
- The token doesn't represent what it claims to

NFATs make the non-fungibility explicit:
- Each position is clearly unique
- Transfers are natural (new holder inherits the deal)
- Secondary markets can price deals individually
- No pretense of fungibility

### Why Offchain Terms?

Putting all deal terms onchain would:
- Increase gas costs significantly
- Reduce flexibility for complex arrangements
- Require contract upgrades for new term types

Offchain terms with onchain custody provides:
- Maximum flexibility for bespoke arrangements
- Simple, auditable onchain contracts
- Easy extension to new deal structures

---

## Wrapped NFATs (Fractionalization)

An NFAT can optionally be wrapped in a fungible ERC-20 token, allowing the deal position to be split up and sold in pieces.

### How It Works

- The NFAT holder deposits their NFAT into a wrapper contract
- The wrapper mints fungible tokens representing fractional ownership of that specific NFAT
- Fractional token holders can trade their shares freely
- When the NFAT receives rewards or is redeemed, proceeds are distributed proportionally to fractional holders

### Use Cases

- **Liquidity**: NFAT holder wants partial liquidity without selling the entire position
- **Syndication**: Multiple parties want exposure to a single deal
- **Smaller denominations**: Large deals can be broken into accessible pieces
- **Secondary markets**: Fungible tokens are easier to trade on existing DEX infrastructure

### Key Characteristics

- Each wrapped NFAT has its own separate fungible token (not pooled across deals)
- The wrapper contract becomes the NFAT holder and receives all rewards/redemptions
- Fractional holders claim their proportional share from the wrapper
- Wrapping is optional — NFATs work fine without it

### Relationship to NFATS

| Aspect | Unwrapped NFAT | Wrapped NFAT |
|--------|----------------|--------------|
| **Ownership** | Single holder | Multiple fractional holders |
| **Transferability** | Whole position only | Fractional shares tradeable |
| **Rewards** | Direct to holder | Via wrapper to fractional holders |
| **Complexity** | Simple | Additional wrapper contract |

### Implementation Notes

- Wrapper is a separate contract, not part of core NFATS
- Can be deployed per-NFAT or as a factory pattern
- Reward claiming and redemption flows need to account for wrapper as intermediary
- Transfer restrictions on the underlying NFAT (if any) apply to the wrapper depositing/withdrawing

---

*Document Version: 0.1*
