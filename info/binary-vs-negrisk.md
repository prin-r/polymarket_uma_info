# Binary vs NegRisk Markets: Complete Comparison

## Overview

Polymarket uses two distinct market architectures depending on whether multiple outcomes can be true simultaneously or only one outcome can win.

---

## Quick Comparison Table

| Feature | Binary Markets | NegRisk Markets |
|---------|----------------|-----------------|
| **Question Type** | Independent outcomes | Mutually exclusive outcomes |
| **Winners** | Multiple can be YES | Only ONE can win |
| **Probability Sum** | Can exceed 100% | MUST equal 100% |
| **Market Structure** | Separate markets | Single unified market |
| **Contract Used** | CTF Exchange | NegRisk CTF Exchange + Fee Module |
| **Capital Efficiency** | No token conversion | NO → YES conversion |
| **QuestionIDs** | One per market | One for entire event |
| **ConditionIDs** | One per market | One with multiple outcome slots |
| **Token IDs** | 2 per market (YES/NO) | 2 × N outcomes |
| **Use Case** | "Which companies will do X?" | "Which company will win?" |

---

## Binary Markets

### Structure

```
Event: "Which DCMs will self-certify by March 31, 2026?"

Market 1: "Will ForecastEx self-certify?" (YES/NO)
  ├── questionId₁ = keccak256(ancillaryData₁)
  ├── conditionId₁ = keccak256(oracle, questionId₁, 2)
  ├── YES token₁ (ERC1155 tokenId)
  └── NO token₁ (ERC1155 tokenId)

Market 2: "Will Railbird self-certify?" (YES/NO)
  ├── questionId₂ = keccak256(ancillaryData₂)
  ├── conditionId₂ = keccak256(oracle, questionId₂, 2)
  ├── YES token₂ (ERC1155 tokenId)
  └── NO token₂ (ERC1155 tokenId)

... (6 more separate markets)
```

### Characteristics

**Multiple Winners Possible:**
- ForecastEx: 85% YES ✓
- Railbird: 55% YES ✓
- Small Exchange: 53% YES ✓
- **Result: All 3+ can resolve to YES**

**Probability Sum:**
- Can exceed 100% (85% + 55% + 53% + ... = possible 200%+)
- No mathematical constraint because outcomes are independent

**Contract Flow:**
```
For each market separately:
1. UmaCtfAdapter.initialize(questionData₁)
2. CTF.prepareCondition(oracle, questionId₁, 2)
3. OptimisticOracle.requestPrice(questionId₁)
```

**UI Presentation:**
- Polymarket groups related markets for UX
- But on-chain: completely separate
- Each has independent oracle resolution

---

## NegRisk Markets

### Structure

```
Event: "Who will win the 2026 FIFA World Cup?"

Single Market with 47 mutually exclusive outcomes:
  ├── questionId = keccak256(ancillaryData)
  ├── conditionId = keccak256(oracle, questionId, 47)
  ├── Outcome 0: Spain (YES token₀, NO token₀)
  ├── Outcome 1: England (YES token₁, NO token₁)
  ├── Outcome 2: France (YES token₂, NO token₂)
  ├── Outcome 3: Argentina (YES token₃, NO token₃)
  └── ... (43 more outcomes)
```

### Characteristics

**Only One Winner:**
- Spain: 16%
- England: 14%
- France: 13%
- Argentina: 11%
- **Result: ONLY ONE resolves to YES, all others NO**

**Probability Sum:**
- MUST equal 100% (16% + 14% + 13% + 11% + ... = 100%)
- Mathematically enforced by market structure
- Arbitrage keeps prices aligned

**Contract Flow:**
```
Single initialization for all outcomes:
1. UmaCtfAdapter.initialize(questionData with 47 outcomes)
2. CTF.prepareCondition(oracle, questionId, 47)
3. OptimisticOracle.requestPrice(questionId)
4. NegRiskAdapter enables NO → YES conversion
```

**Capital Efficiency:**

Traditional approach (without NegRisk):
```
To bet on Spain and hedge:
├── Buy YES on Spain: 1 USDC
├── Buy NO on England: 1 USDC
├── Buy NO on France: 1 USDC
└── ... (46 times)
Total: 47 USDC required
```

With NegRisk conversion:
```
1 NO token for Spain converts to:
├── 1 YES token for England
├── 1 YES token for France
├── 1 YES token for Argentina
└── ... (all 46 other outcomes)

Market makers can efficiently hedge with less capital!
```

---

## Why NegRisk is Necessary

### Problem: Binary Markets for Winner-Take-All Events

If you used 47 separate binary markets for World Cup:

#### Issue 1: Probability Inconsistency
```
Binary approach:
├── Spain: 20% YES
├── England: 18% YES
├── France: 15% YES
└── Sum: Could be 120% or 80%

But exactly ONE team wins!
Probabilities MUST sum to 100%
```

#### Issue 2: Arbitrage Opportunities

**If Sum > 100%:**
```
Buy NO on all 47 markets
Cost: 47 × $0.40 = $18.80
Payout: 46 markets × $1.00 = $46.00
Guaranteed profit: $27.20
```

**If Sum < 100%:**
```
Buy YES on all 47 markets
Cost: 47 × $0.15 = $7.05
Payout: 1 market × $1.00 = $1.00
Wait... you lose money!

This creates pricing chaos
```

#### Issue 3: Market Maker Impossibility

Without NegRisk:
- Must provide liquidity for 47 separate markets
- Need 47× the capital
- Can't efficiently hedge positions
- Spreads would be massive

With NegRisk:
- Single unified liquidity pool
- Convert NO → YES across markets
- Efficient hedging
- Tight spreads

---

## Contract Differences

### Binary Market Contracts

```
User → UmaCtfAdapter → CTF + OptimisticOracle
User → CTF Exchange → matchOrders()
```

**Contracts:**
- UmaCtfAdapter: `0x2F5e3684cb1F318ec51b00Edba38d79Ac2c0aA9d`
- CTF: `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045`
- CTF Exchange: `0x4bFb41d5B3570DeFd03C39A9A4D8dE6Bd8B8982E`

### NegRisk Market Contracts

```
User → UmaCtfAdapter → CTF + OptimisticOracle
User → NegRisk Adapter → Convert NO to YES
User → NegRisk CTF Exchange → matchOrders()
         └─> NegRisk Fee Module 2 → Fee handling
```

**Contracts:**
- UmaCtfAdapter: `0x2F5e3684cb1F318ec51b00Edba38d79Ac2c0aA9d` (same)
- CTF: `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045` (same)
- NegRisk CTF Exchange: `0xC5d563A36AE78145C45a50134d48A1215220F80a`
- NegRisk Adapter: `0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296`
- NegRisk Fee Module 2: `0xB768891e3130F6dF18214Ac804d4DB76c2C37730`

---

## Resolution Differences

### Binary Market Resolution

Each market resolves independently:

```
Market 1 (ForecastEx):
├── ProposePrice(1e18) → YES
└── PayoutVector: [1, 0]

Market 2 (Railbird):
├── ProposePrice(1e18) → YES
└── PayoutVector: [1, 0]

Market 3 (Small Exchange):
├── ProposePrice(0) → NO
└── PayoutVector: [0, 1]

Result: Multiple YES outcomes possible
```

### NegRisk Market Resolution

Single resolution for all outcomes:

```
World Cup Event:
├── ProposePrice(3) → France wins (index 2)
└── PayoutVector: [0, 0, 1, 0, 0, ..., 0]
                   ↑     ↑  ↑
                 Spain France Argentina
                  NO   YES   NO

Only ONE outcome gets payout [1]
All others get payout [0]
```

---

## Real Examples

### Binary: DCMs Self-Certification Event

**URL:** https://polymarket.com/event/which-dcms-self-certify-sports-event-contracts-by-march-31-2026

**Structure:**
- 8 separate binary markets
- Each can independently be YES or NO
- Probabilities: 85%, 55%, 53%, 51%, 51%, 48%, 47%, 50%
- Sum: ~440% (valid because multiple can happen)

**Why Binary?**
Multiple companies can self-certify. This is NOT mutually exclusive.

### NegRisk: 2026 FIFA World Cup Winner

**URL:** https://polymarket.com/event/2026-fifa-world-cup-winner-595

**Structure:**
- Single market with 47 outcomes
- Only ONE team can win
- Probabilities: 16%, 14%, 13%, 11%, 10%, 8%, 6%, ...
- Sum: 100% (enforced)
- Volume: $71.3M
- `negRisk: true` flag

**Why NegRisk?**
Only one team can win the World Cup. Mutually exclusive outcomes.

---

## Token ID Calculation

### Binary Markets

Each market has 2 token IDs:

```solidity
// For Market 1
conditionId₁ = keccak256(oracle, questionId₁, 2)
yesTokenId₁ = keccak256(USDC, keccak256(conditionId₁, indexSet=1))
noTokenId₁ = keccak256(USDC, keccak256(conditionId₁, indexSet=2))

// For Market 2
conditionId₂ = keccak256(oracle, questionId₂, 2)
yesTokenId₂ = keccak256(USDC, keccak256(conditionId₂, indexSet=1))
noTokenId₂ = keccak256(USDC, keccak256(conditionId₂, indexSet=2))

Total: 16 token IDs (8 markets × 2)
```

### NegRisk Markets

Single conditionId with multiple outcomes:

```solidity
conditionId = keccak256(oracle, questionId, 47)

// Outcome 0: Spain
spainYesTokenId = keccak256(USDC, keccak256(conditionId, indexSet=0b1))
spainNoTokenId = keccak256(USDC, keccak256(conditionId, indexSet=0b1111...110))

// Outcome 1: England
englandYesTokenId = keccak256(USDC, keccak256(conditionId, indexSet=0b10))
englandNoTokenId = keccak256(USDC, keccak256(conditionId, indexSet=0b1111...101))

Total: 94 token IDs (47 outcomes × 2)
```

---

## Trading Flow Comparison

### Binary Market Trade

```
User buys YES on "Will ForecastEx self-certify?"

1. User creates signed order:
   - makerAssetId: ForecastEx YES tokenId
   - takerAssetId: 0 (USDC)
   - makerAmount: 100
   - price: 0.85 USDC

2. CTF Exchange matches order
3. USDC transfer: User → Counterparty
4. Token transfer: Counterparty → User
```

### NegRisk Market Trade

```
User buys YES on "Spain wins World Cup"

1. User creates signed order:
   - makerAssetId: Spain YES tokenId
   - takerAssetId: 0 (USDC)
   - makerAmount: 100
   - price: 0.16 USDC

2. NegRisk CTF Exchange matches order
3. NegRisk Fee Module 2 handles fees
4. USDC transfer: User → Counterparty
5. Token transfer: Counterparty → User

BONUS: User can later convert NO tokens
6. User has NO for Spain (if they hedge)
7. NegRisk Adapter converts:
   1 NO (Spain) → 1 YES (England + France + ... all 46 others)
```

---

## When to Use Each Type

### Use Binary Markets When:

✅ Multiple outcomes can be true simultaneously
✅ Outcomes are independent events
✅ Probabilities can logically exceed 100%

**Examples:**
- "Which companies will launch X by date Y?"
- "Will [list of events] happen?"
- "Which countries will reach GDP threshold?"

### Use NegRisk Markets When:

✅ Only ONE outcome can be true
✅ Outcomes are mutually exclusive
✅ Probabilities MUST sum to 100%

**Examples:**
- Elections: "Who will win?"
- Sports: "Which team will win the championship?"
- Awards: "Who will win Best Picture?"
- Rankings: "Who will finish #1?"

---

## Key Takeaways

1. **Binary markets** = Separate independent markets grouped by UI
2. **NegRisk markets** = Single market with multiple mutually exclusive outcomes
3. **NegRisk is NOT optional** for winner-take-all events (would be arbitraged immediately)
4. **Token conversion** in NegRisk provides capital efficiency for market makers
5. **Same initialization contract** (UmaCtfAdapter) for both types
6. **Different trading contracts** (CTF Exchange vs NegRisk CTF Exchange)

---

## Source Code References

### Key Contract Functions

| Function | Contract | Source |
|----------|----------|--------|
| `splitPosition()` | CTF | [ConditionalTokens.sol#L105-L159](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol#L105-L159) |
| `prepareCondition()` | CTF | [ConditionalTokens.sol#L67-L83](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol#L67-L83) |
| `initialize()` | UmaCtfAdapter | [UmaCtfAdapter.sol#L125-L167](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L125-L167) |
| `matchOrders()` | CTF Exchange | [Trading.sol](https://github.com/Polymarket/ctf-exchange/blob/main/src/exchange/mixins/Trading.sol) |

---

## References

- [Polymarket NegRisk Documentation](https://docs.polymarket.com/developers/neg-risk/overview)
- [NegRisk Adapter GitHub](https://github.com/Polymarket/neg-risk-ctf-adapter)
- [CTF Exchange GitHub](https://github.com/Polymarket/ctf-exchange)
- [Exchange Fee Module GitHub](https://github.com/Polymarket/exchange-fee-module)
- [NegRisk Market Rebalancing Analysis](https://medium.com/@navnoorbawa/negrisk-market-rebalancing-how-29m-was-extracted-from-multi-condition-prediction-markets-2f1f91644c5b)
