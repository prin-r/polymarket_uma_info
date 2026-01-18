# Polymarket Terminology

> Glossary of terms used in Polymarket and prediction market infrastructure


---

## A

### Ancillary Data
Bytes-encoded metadata attached to a UMA price request. Contains the question text, resolution rules, and market metadata. Used to derive the `questionId` via keccak256 hash.

```
Example: "q: Will Bitcoin reach $100,000 by Dec 31, 2025? res_data: p1:0, p2:1..."
```

### Assertion
A claim made by a proposer about the outcome of an event. In UMA's system, assertions are optimistically accepted unless disputed within the liveness period.

---

## B

### Binary Market
A prediction market with exactly two mutually exclusive outcomes: YES and NO. The sum of YES and NO token prices should equal ~$1.00.

### Bond
Collateral (USDC) that proposers and disputers must lock when participating in the resolution process. Default is ~750 USDC on Polymarket. Lost if the party is proven wrong; returned plus reward if correct.

### Bulletin Board
A feature in UmaCtfAdapter v2+ allowing market creators to issue clarifications about resolution rules without changing the core question. Address: `0x6A9D222616C90FcA5754cd1333cFD9b7fb6a4F74`.

---

## C

### Challenge Period
See **Liveness Period**.

### CLOB (Central Limit Order Book)
Polymarket's hybrid order book system. Orders are matched off-chain by an operator, but settlement happens on-chain non-custodially. Sometimes called BLOB (Binary Limit Order Book).

### Collection ID
A bytes32 identifier for a specific outcome collection in CTF. Derived from `parentCollectionId`, `conditionId`, and `indexSet`. Used to calculate Position IDs.

```solidity
collectionId = CTF.getCollectionId(parentCollectionId, conditionId, indexSet)
```

### Collateral
The underlying asset backing outcome tokens. On Polymarket, this is USDC.e (bridged USDC) on Polygon.

### Commit-Reveal Voting
UMA's two-phase voting mechanism for DVM disputes:
1. **Commit phase**: Voters submit hash(vote + salt) to hide their vote
2. **Reveal phase**: Voters reveal actual vote and salt for verification

### Condition
A prepared question in the CTF system with defined outcomes. Each condition has a unique `conditionId` and represents a single prediction market.

### Condition ID
A bytes32 identifier for a market condition. Derived from the oracle address, question ID, and outcome count.

```solidity
conditionId = keccak256(abi.encodePacked(oracle, questionId, outcomeSlotCount))
```

### Conditional Tokens Framework (CTF)
Gnosis's protocol for creating tokenized prediction market outcomes. Implements ERC1155 standard. Contract: `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045`.

**Source:** [ConditionalTokens.sol](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol)

### Convert
In NegRisk markets, the action of exchanging NO tokens from one outcome into YES tokens for all other outcomes. Enables capital efficiency in multi-outcome markets.

---

## D

### Deep Position
A position in nested conditional markets where multiple conditions are combined. Example: "YES on condition A AND YES on condition B". Polymarket primarily uses single-level (shallow) positions.

### Disputer
A participant who challenges a proposed resolution by posting a counter-bond. If correct, they receive their bond back plus a portion of the proposer's bond.

### DVM (Data Verification Mechanism)
UMA's fallback arbitration system. When disputes escalate twice, UMA token holders vote on the correct outcome over 48-96 hours. The "supreme court" of the oracle system.

**Source:** [VotingV2.sol](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/data-verification-mechanism/implementation/VotingV2.sol)

---

## E

### EIP-712
Ethereum standard for typed structured data signing. Used by Polymarket for order signatures, enabling off-chain order creation with on-chain verification.

### Emergency Resolution
Admin function allowing Polymarket to resolve markets outside the normal UMA flow in case of oracle failures or invalid outcomes (like double-YES in NegRisk).

### ERC1155
Multi-token standard used by CTF. Allows multiple token types (YES/NO outcomes) to be managed by a single contract. Each outcome has a unique `tokenId` (Position ID).

### Escalation Game
The mechanism by which UMA's Optimistic Oracle works. Proposals are assumed correct unless challenged, with disputes escalating to progressively higher arbitration levels.

---

## F

### Final Fee
A protocol fee charged by UMA for processing disputes through the DVM. Added on top of the proposal bond.

### Finder
UMA's registry contract that stores addresses of other protocol contracts. Used to look up OptimisticOracle, Store, and other contract addresses.

---

## G

### GAT (Global Activity Threshold)
Minimum number of UMA tokens that must participate in a DVM vote for it to be valid.

---

## H

### Hybrid-Decentralized
Polymarket's architecture model where:
- **Off-chain**: Order matching and book management
- **On-chain**: Non-custodial settlement and token custody

---

## I

### Index Set
A 256-bit array indicating which outcome slots are included in a position. For binary markets:
- `1` (0b01) = YES (first outcome)
- `2` (0b10) = NO (second outcome)
- `3` (0b11) = Both outcomes

### Invalid Outcome
When a market cannot be properly resolved (ambiguous question, event didn't occur as expected). Results in `[1,1]` payout where both YES and NO tokens redeem for $0.50.

---

## L

### Liveness Period
The time window during which a proposed resolution can be disputed. Default is 7,200 seconds (2 hours) on Polymarket. Also called "challenge period" or "dispute window".

---

## M

### Maker
The party who creates an order and waits for it to be filled. In Polymarket's CLOB, makers provide liquidity to the order book.

### Market Creator
A whitelisted address authorized to initialize new markets on Polymarket via the UmaCtfAdapter.

### Merge
The action of combining equal amounts of all outcome tokens back into collateral. Opposite of split.

```solidity
// Burn 100 YES + 100 NO → Receive 100 USDC
CTF.mergePositions(USDC, 0x0, conditionId, [1,2], 100e6)
```

### MINT Mode
Exchange matching mode where complementary buy orders (YES + NO) are matched by minting new tokens from collateral.

### MOOV2 (Managed Optimistic Oracle V2)
Updated UMA oracle (UMIP-189, Aug 2025) that restricts proposal rights to whitelisted addresses while keeping disputes open to anyone. Improves proposal quality.

---

## N

### NegRisk (Negative Risk)
Architecture for multi-outcome markets where exactly one outcome wins. Allows capital-efficient trading by enabling conversion between NO positions across different outcomes.

### NO Token
The outcome token representing the "No" or second outcome in a binary market. Index set = 2.

### Nonce
A counter used for order management. Incrementing your nonce invalidates all orders signed with the previous nonce, enabling bulk cancellation.

---

## O

### Operator
The off-chain service that manages Polymarket's order book, matches orders, and submits matched trades for on-chain settlement. Cannot execute without valid user signatures.

### Optimistic Oracle
UMA's oracle system that assumes proposed data is correct unless challenged. Enables fast resolution (~2 hours) for uncontested outcomes while maintaining security through economic incentives.

**Source:** [OptimisticOracleV2.sol](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol)

### Oracle
A system that brings off-chain data on-chain. In Polymarket, the UmaCtfAdapter serves as the oracle for CTF conditions, fetching resolution data from UMA's Optimistic Oracle.

### Outcome Slot
One possible result of a condition. Binary markets have 2 outcome slots (YES and NO). CTF supports up to 256 outcome slots.

---

## P

### Partition
An array of disjoint index sets used in split/merge operations. For binary markets, the standard partition is `[1, 2]` (YES and NO separately).

### Payout Denominator
The sum of all payout numerators for a condition. Used to calculate proportional redemption amounts.

### Payout Numerator
The value assigned to each outcome after resolution. Common patterns:
- `[1, 0]` = YES wins
- `[0, 1]` = NO wins
- `[1, 1]` = Invalid (50/50 split)

### Payout Vector
The array of payout numerators reported by the oracle. Determines how much collateral each outcome token can redeem for.

### Position
A holding of conditional tokens representing a stake in one or more outcomes.

### Position ID
The ERC1155 token ID for a specific outcome position. Derived from collateral token and collection ID.

```solidity
positionId = CTF.getPositionId(collateralToken, collectionId)
```

### Price Identifier
A registered identifier in UMA's system specifying how to resolve a type of question. Polymarket uses:
- `YES_OR_NO_QUERY` (UMIP-107) for binary questions
- `MULTIPLE_VALUES` (UMIP-183) for multi-value responses

### Proposal
A suggested resolution submitted to the Optimistic Oracle. Must include a bond and specifies the proposed outcome (YES, NO, or invalid).

### Proposer
A participant who submits a resolution proposal to the Optimistic Oracle. Must post a bond (~750 USDC). Earns reward (~5 USDC) if correct and not disputed.

---

## Q

### Question ID
A bytes32 identifier derived from the ancillary data hash. Used to reference a specific question across contracts.

```solidity
questionId = keccak256(ancillaryData)
```

---

## R

### Redeem
The action of exchanging winning outcome tokens for collateral after market resolution.

```solidity
CTF.redeemPositions(USDC, 0x0, conditionId, [1]) // Redeem YES tokens
```

### Report Payouts
The oracle function that finalizes a condition's outcome by setting the payout vector.

```solidity
CTF.reportPayouts(questionId, [1, 0]) // YES wins
```

### Request
A price/data request submitted to the Optimistic Oracle. Created when a market is initialized.

### Reset
When a first dispute occurs, the market is reset and a new proposal can be submitted. The original proposer and disputer wait for final resolution.

### Resolution
The process of determining and recording the outcome of a prediction market.

---

## S

### Schelling Point
A solution people tend to choose without communication. UMA's DVM relies on voters converging on the "obvious" correct answer.

### Settlement
The on-chain execution of matched orders, transferring tokens and/or collateral between parties.

### Slashing
The penalty mechanism where incorrect or inactive DVM voters lose a portion of their staked UMA tokens, which is redistributed to correct voters.

### SPAT (Schelling Point Agreement Threshold)
Minimum percentage (50%) of staked UMA tokens that must agree for a DVM vote to resolve.

### Split
The action of converting collateral into a complete set of outcome tokens.

```solidity
// Deposit 100 USDC → Receive 100 YES + 100 NO
CTF.splitPosition(USDC, 0x0, conditionId, [1,2], 100e6)
```

### Store
UMA contract that manages fees and tracks final fee amounts for different currencies.

---

## T

### Taker
The party who fills an existing order. In matched trades, takers receive any price improvement.

### Token ID
See **Position ID**.

---

## U

### UMIP (UMA Improvement Proposal)
Governance proposals for UMA protocol changes. Relevant ones:
- UMIP-107: YES_OR_NO_QUERY identifier
- UMIP-183: MULTIPLE_VALUES identifier
- UMIP-189: Managed Optimistic Oracle V2

### UmaCtfAdapter
Polymarket's bridge contract between UMA's Optimistic Oracle and Gnosis CTF. Handles market initialization, resolution requests, and payout reporting.

**Source:** [UmaCtfAdapter.sol](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol)

---

## W

### Whitelisted Proposer
Under MOOV2, only approved addresses can submit resolution proposals. Includes Risk Labs/Polymarket employees and users with proven track records (20+ proposals, >95% accuracy).

### Wrapped Collateral
In NegRisk architecture, USDC is wrapped into an ERC20 token to enable conversion operations between markets.

---

## Y

### YES Token
The outcome token representing the "Yes" or first outcome in a binary market. Index set = 1.

### YES_OR_NO_QUERY
UMA price identifier (UMIP-107) for binary questions. Returns:
- `1e18` (1 × 10¹⁸) = YES
- `0` = NO
- `0.5e18` = Invalid/Unknown

---

## Numeric

### 7200
Default liveness period in seconds (2 hours).

### 750 USDC
Approximate default bond amount for proposers and disputers.

### 137
Polygon Mainnet chain ID.

---

## Contract Quick Reference

| Term | Address |
|------|---------|
| CTF | `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045` |
| USDC.e | `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` |
| OptimisticOracleV2 | `0xee3afe347d5c74317041e2618c49534daf887c24` |
| UmaCtfAdapter v3 | `0x2F5e3684cb1F318ec51b00Edba38d79Ac2c0aA9d` |
| CTF Exchange | `0x4bFb41d5B3570DeFd03C39A9A4D8dE6Bd8B8982E` |
| NegRisk Adapter | `0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296` |

---

## Source Code References

| Contract | GitHub Repository |
|----------|-------------------|
| ConditionalTokens | [gnosis/conditional-tokens-contracts](https://github.com/gnosis/conditional-tokens-contracts) |
| OptimisticOracleV2 | [UMAprotocol/protocol](https://github.com/UMAprotocol/protocol/tree/master/packages/core/contracts/optimistic-oracle-v2) |
| UmaCtfAdapter | [Polymarket/uma-ctf-adapter](https://github.com/Polymarket/uma-ctf-adapter) |
| CTF Exchange | [Polymarket/ctf-exchange](https://github.com/Polymarket/ctf-exchange) |
| NegRisk Adapter | [Polymarket/neg-risk-ctf-adapter](https://github.com/Polymarket/neg-risk-ctf-adapter) |
