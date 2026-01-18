# Polymarket Documentation

Comprehensive technical documentation for Polymarket's prediction market architecture on Polygon.


---

## Quick Start

New to Polymarket? Start here:

1. [Architecture Overview](architecture.md) - High-level system design
2. [Terminology](terminology.md) - Essential terms and definitions
3. [Market Lifecycle](market-lifecycle.md) - How markets work from creation to redemption

---

## Documentation Index

### Core Concepts

| Document | Description | Best For |
|----------|-------------|----------|
| [architecture.md](architecture.md) | System overview, contract relationships | Understanding the big picture |
| [terminology.md](terminology.md) | Glossary of 50+ Polymarket terms | Quick reference, learning concepts |
| [binary-vs-negrisk.md](binary-vs-negrisk.md) | Binary vs NegRisk market comparison | Understanding market types |

### Smart Contracts

| Document | Description | Best For |
|----------|-------------|----------|
| [contracts.json](contracts.json) | All contract addresses and versions | Finding contract addresses |
| [conditional-tokens.md](conditional-tokens.md) | Gnosis CTF | Understanding outcome tokens |
| [uma-oracle.md](uma-oracle.md) | UMA Optimistic Oracle mechanics | Understanding resolution |

### Market Operations

| Document | Description | Best For |
|----------|-------------|----------|
| [market-creation.md](market-creation.md) | Contract-level market creation walkthrough | Creating markets, debugging |
| [market-lifecycle.md](market-lifecycle.md) | Complete market lifecycle steps | Trading, redemption |
| [clob-exchange.md](clob-exchange.md) | Exchange mechanics and order matching | Understanding trading |
| [resolution-qa.md](resolution-qa.md) | Q&A on resolution flow, proposers, disputes | Understanding resolution process |

---

## Key Topics

### Market Creation

**Start:** [market-creation.md](market-creation.md)

Learn how to create a Polymarket prediction market:
- Contract function calls with code snippets
- Real transaction examples from PolygonScan
- Token minting mechanics
- Initial pricing (hint: it's NOT hardcoded!)

**Key Questions Answered:**
- When are tokens minted? (Not during initialization!)
- What's the initial price? (Determined by orderbook, typically ~$0.50)
- What gets saved on-chain? (questionId, conditionId, metadata - but NO tokens or prices)

### Binary vs NegRisk Markets

**Start:** [binary-vs-negrisk.md](binary-vs-negrisk.md)

Understand the two market types:

**Binary Markets:**
- Multiple outcomes can be true
- Example: "Which companies will do X by date Y?"
- Separate questionIds for each market
- Uses CTF Exchange

**NegRisk Markets:**
- Only ONE outcome can win
- Example: "Who will win the World Cup?"
- Single questionId with multiple outcome slots
- Uses NegRisk CTF Exchange + Adapter
- NO tokens convert to YES tokens (capital efficiency!)

### Trading Flow

**Start:** [clob-exchange.md](clob-exchange.md)

How trading actually works:
- Off-chain orderbook (CLOB API)
- On-chain settlement (CTF Exchange)
- MINT mode: Auto-minting from complementary orders
- MERGE mode: Burning complete sets back to USDC

### Resolution & Disputes

**Start:** [uma-oracle.md](uma-oracle.md) or [resolution-qa.md](resolution-qa.md)

How markets resolve:
- Optimistic Oracle mechanics
- 2-hour liveness period
- Dispute process (~750 USDC bond)
- DVM escalation for disputes
- Payout vectors: `[1,0]`, `[0,1]`, `[1,1]`

**Common Questions:** [resolution-qa.md](resolution-qa.md)
- Who requests resolution from UMA?
- How does CTF get the answer?
- What happens on disputes?
- Why are bonds burned?
- Who are the 3rd party proposers?
- Can you resolve immediately after creation?

---

## Contract Addresses (Polygon Mainnet)

### Core Infrastructure

| Contract | Address | Purpose |
|----------|---------|---------|
| **CTF** | `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045` | Gnosis Conditional Tokens |
| **UmaCtfAdapter v3.1** | `0x157Ce2d672854c848c9b79C49a8Cc6cc89176a49` | Latest market creation adapter |
| **UmaCtfAdapter v3** | `0x2F5e3684cb1F318ec51b00Edba38d79Ac2c0aA9d` | Primary production adapter (179k+ tx) |
| **OptimisticOracleV2** | `0xee3afe347d5c74317041e2618c49534daf887c24` | UMA oracle for resolution |
| **USDC.e** | `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` | Collateral token |

### Trading

| Contract | Address | Purpose |
|----------|---------|---------|
| **CTF Exchange** | `0x4bFb41d5B3570DeFd03C39A9A4D8dE6Bd8B8982E` | Binary market trading |
| **NegRisk CTF Exchange** | `0xC5d563A36AE78145C45a50134d48A1215220F80a` | Multi-outcome trading |
| **NegRisk Adapter** | `0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296` | NO → YES conversion |
| **NegRisk Fee Module 2** | `0xB768891e3130F6dF18214Ac804d4DB76c2C37730` | Fee handling |

**Full list:** See [contracts.json](contracts.json)

---

## Common Questions

### "How do I create a market?"

See: [market-creation.md](market-creation.md)

```solidity
1. Approve USDC to UmaCtfAdapter
2. Call UmaCtfAdapter.initialize(...)
   - Creates questionId and conditionId
   - Registers with OptimisticOracle
   - NO tokens minted yet!
3. Market makers post initial orders off-chain
4. First trade triggers token minting via CTF.splitPosition()
```

### "What's the difference between YES and NO tokens?"

They're completely different ERC1155 tokens with different token IDs:
- `yesTokenId = keccak256(USDC, keccak256(conditionId, indexSet=1))`
- `noTokenId = keccak256(USDC, keccak256(conditionId, indexSet=2))`

At resolution, only the winning token redeems for 1 USDC.

### "Why use NegRisk for elections but not for company events?"

Elections: Only ONE candidate wins → NegRisk required
- Probabilities MUST sum to 100%
- Without NegRisk, arbitrage would break the market

Company events: Multiple can succeed → Binary markets work fine
- Probabilities can exceed 100% (that's valid!)
- Each market resolves independently

See: [binary-vs-negrisk.md](binary-vs-negrisk.md)

### "When are outcome tokens minted?"

NOT during market creation! Tokens are minted when:
1. Trader manually calls `CTF.splitPosition()`, OR
2. Exchange auto-mints in MINT mode (complementary orders)

See: [market-creation.md#token-minting--initial-pricing](market-creation.md#token-minting--initial-pricing)

### "What's the initial price?"

There is NO hardcoded initial price. Prices come from:
- Market makers posting initial orderbook (off-chain)
- Typically around $0.50/$0.50 for 50% probability
- But can start anywhere based on information

---

## External Resources

### Official Links
- [Polymarket Documentation](https://docs.polymarket.com/)
- [Polymarket Website](https://polymarket.com)
- [PolygonScan](https://polygonscan.com/)

### GitHub Repositories & Source Code

| Repository | Key Files |
|------------|-----------|
| [UMA CTF Adapter](https://github.com/Polymarket/uma-ctf-adapter) | [UmaCtfAdapter.sol](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol) |
| [CTF Exchange](https://github.com/Polymarket/ctf-exchange) | [CTFExchange.sol](https://github.com/Polymarket/ctf-exchange/blob/main/src/exchange/CTFExchange.sol), [Trading.sol](https://github.com/Polymarket/ctf-exchange/blob/main/src/exchange/mixins/Trading.sol) |
| [NegRisk Adapter](https://github.com/Polymarket/neg-risk-ctf-adapter) | [NegRiskAdapter.sol](https://github.com/Polymarket/neg-risk-ctf-adapter/blob/main/src/NegRiskAdapter.sol) |
| [Exchange Fee Module](https://github.com/Polymarket/exchange-fee-module) | Fee calculation logic |
| [Gnosis CTF](https://github.com/gnosis/conditional-tokens-contracts) | [ConditionalTokens.sol](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol), [CTHelpers.sol](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/CTHelpers.sol) |
| [UMA Protocol](https://github.com/UMAprotocol/protocol) | [OptimisticOracleV2.sol](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol), [VotingV2.sol](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/data-verification-mechanism/implementation/VotingV2.sol) |

### Learning Resources
- [UMA Docs](https://docs.uma.xyz/)
- [Gnosis Conditional Tokens Docs](https://docs.gnosis.io/conditionaltokens/)
- [UMIP-107: YES_OR_NO_QUERY](https://github.com/UMAprotocol/UMIPs/blob/master/UMIPs/umip-107.md)

---

## File Structure

```
blockchain/polymarket/
├── README.md                        # This file
├── architecture.md                  # System overview
├── terminology.md                   # Glossary (50+ terms)
├── binary-vs-negrisk.md            # Market type comparison
├── contracts.json                   # All contract addresses
├── market-creation.md    # In-depth market creation guide
├── market-lifecycle.md              # Complete lifecycle walkthrough
├── conditional-tokens.md            # Gnosis CTF
├── uma-oracle.md                    # Oracle mechanics
├── clob-exchange.md                # Trading mechanics
└── resolution-qa.md                 # Q&A on resolution, disputes, proposers
```

---

## Contributing

This documentation is based on:
- On-chain transaction analysis
- Official Polymarket/UMA/Gnosis documentation
- GitHub source code review
- Real-world usage patterns

Last verified: 2026-01-18

---

## Quick Reference: Market Creation Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                     MARKET CREATION FLOW                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. APPROVE USDC                                                │
│     └─> USDC.approve(UmaCtfAdapter, reward)                     │
│                                                                 │
│  2. INITIALIZE MARKET                                           │
│     └─> UmaCtfAdapter.initialize(...)                           │
│         ├─> CTF.prepareCondition() → conditionId created        │
│         ├─> OptimisticOracle.requestPrice() → oracle request    │
│         └─> QuestionData saved                                  │
│                                                                 │
│  3. MARKET MAKERS POST ORDERS (off-chain)                       │
│     └─> CLOB API                                                │
│                                                                 │
│  4. FIRST TRADE MINTS TOKENS                                    │
│     └─> Exchange.matchOrders()                                  │
│         └─> CTF.splitPosition() → YES/NO tokens minted          │
│                                                                 │
│  5. PROPOSE RESOLUTION                                          │
│     └─> OptimisticOracle.proposePrice(answer, bond)             │
│                                                                 │
│  6. LIVENESS PERIOD (2 hours)                                   │
│     └─> Can be disputed with ~750 USDC bond                     │
│                                                                 │
│  7. RESOLVE MARKET                                              │
│     └─> UmaCtfAdapter.resolve(questionId)                       │
│         └─> CTF.reportPayouts([1,0] or [0,1])                   │
│                                                                 │
│  8. REDEEM WINNINGS                                             │
│     └─> CTF.redeemPositions() → winning tokens → USDC           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Legend

- ✅ = Happens on-chain
- ❌ = Does NOT happen / Not required
- 🔍 = See detailed documentation
- 📝 = Example code available
- 🔗 = External link
