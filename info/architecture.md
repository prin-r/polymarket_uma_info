# Polymarket Technical Architecture

> Deep dive into Polymarket's prediction market infrastructure on Polygon

**Last Updated:** 2026-01-18
**Network:** Polygon Mainnet (Chain ID: 137)

## Overview

Polymarket is a decentralized prediction market platform built on Polygon. It uses:
- **Gnosis Conditional Tokens Framework (CTF)** for tokenizing outcomes as ERC1155 tokens
- **UMA Optimistic Oracle V2** for decentralized market resolution
- **Hybrid CLOB (Central Limit Order Book)** for order matching and settlement

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         POLYMARKET ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────┐    ┌──────────────────┐    ┌─────────────────────┐    │
│  │   Traders   │───▶│   CLOB (Off-chain)│───▶│  CTF Exchange       │    │
│  └─────────────┘    │   Order Matching  │    │  (On-chain Settle)  │    │
│                     └──────────────────┘    └──────────┬──────────┘    │
│                                                        │               │
│  ┌─────────────┐    ┌──────────────────┐    ┌─────────▼──────────┐    │
│  │   Market    │───▶│  UmaCtfAdapter   │───▶│ Conditional Tokens │    │
│  │   Creator   │    │   (Oracle Glue)   │    │   (ERC1155 CTF)    │    │
│  └─────────────┘    └────────┬─────────┘    └────────────────────┘    │
│                              │                                         │
│                     ┌────────▼─────────┐    ┌─────────────────────┐    │
│                     │  UMA Optimistic  │───▶│   UMA DVM           │    │
│                     │  Oracle V2       │    │   (Dispute Voting)  │    │
│                     └──────────────────┘    └─────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Contract Addresses (Polygon Mainnet)

### Core Infrastructure

| Contract | Address | Description |
|----------|---------|-------------|
| **Conditional Tokens (CTF)** | `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045` | Gnosis ERC1155 outcome tokens |
| **USDC.e (Collateral)** | `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` | Bridged USDC (6 decimals) |

### Polymarket Contracts

| Contract | Address | Notes |
|----------|---------|-------|
| **CTF Exchange** | `0x4bFb41d5B3570DeFd03C39A9A4D8dE6Bd8B8982E` | Binary market settlement |
| **NegRisk CTF Exchange** | `0xC5d563A36AE78145C45a50134d48A1215220F80a` | Multi-outcome market settlement |
| **NegRisk Adapter** | `0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296` | Multi-outcome conversions |
| **UmaCtfAdapter v3** | `0x2F5e3684cb1F318ec51b00Edba38d79Ac2c0aA9d` | Current production adapter |
| **UmaCtfAdapter v3.1** | `0x157Ce2d672854c848c9b79C49a8Cc6cc89176a49` | Latest release |
| **UmaCtfAdapter v2** | `0x6A9D222616C90FcA5754cd1333cFD9b7fb6a4F74` | Legacy (still active) |
| **UmaCtfAdapter v1** | `0xCB1822859cEF82Cd2Eb4E6276C7916e692995130` | Deprecated |

### UMA Protocol Contracts (Polygon)

| Contract | Address |
|----------|---------|
| **OptimisticOracleV2** | `0xee3afe347d5c74317041e2618c49534daf887c24` |
| **OptimisticOracleV3** | `0x5953f2538F613E05bAED8A5AeFa8e6622467AD3D` |
| **Finder** | `0x09aea4b2242abC8bb4BB78D537A67a245A7bEC64` |
| **Store** | `0xE58480CA74f1A819faFd777BEDED4E2D5629943d` |
| **IdentifierWhitelist** | `0x2271a5E74eA8A29764ab10523575b41AA52455f0` |

---

## Market Types

### 1. Binary Markets (Simple YES/NO)
- Single question with two outcomes
- Uses standard CTF Exchange
- Example: "Will X happen by date Y?"

### 2. Multi-Outcome Markets (NegRisk)
- Multiple mutually exclusive outcomes where exactly one wins
- Uses NegRisk Adapter for capital efficiency
- Example: "Who will win the election?" (Candidate A, B, C, or Other)

**NegRisk Capital Efficiency:**
- NO tokens from any market can convert to YES tokens in all other markets
- Reduces capital requirements for market makers
- Requires complete outcome universe (including "Other" option)

---

## References

- [Polymarket Documentation](https://docs.polymarket.com/)
- [Polymarket CTF Overview](https://docs.polymarket.com/developers/CTF/overview)
- [Polymarket Resolution Docs](https://docs.polymarket.com/developers/resolution/UMA)
- [UMA Protocol Docs](https://docs.uma.xyz/)
- [Gnosis Conditional Tokens](https://github.com/gnosis/conditional-tokens-contracts)
- [Polymarket UMA Adapter GitHub](https://github.com/Polymarket/uma-ctf-adapter)
- [Polymarket CTF Exchange GitHub](https://github.com/Polymarket/ctf-exchange)
- [Polymarket NegRisk Adapter GitHub](https://github.com/Polymarket/neg-risk-ctf-adapter)
