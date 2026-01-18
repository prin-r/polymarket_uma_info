# Gnosis Conditional Tokens Framework (CTF)

> ERC1155 tokens representing prediction market outcomes


## Overview

The Conditional Tokens Framework (CTF) is Gnosis's system for creating tokenized conditional outcomes. Polymarket uses CTF to represent binary (YES/NO) outcomes as ERC1155 tokens backed by USDC collateral.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONDITIONAL TOKENS STRUCTURE                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   COLLATERAL (USDC)                                                     │
│         │                                                               │
│         ▼ splitPosition                                                 │
│   ┌─────────────┐                                                       │
│   │  CONDITION  │ ◄── conditionId = hash(oracle, questionId, 2)         │
│   │  (Binary)   │                                                       │
│   └──────┬──────┘                                                       │
│          │                                                              │
│    ┌─────┴─────┐                                                        │
│    ▼           ▼                                                        │
│ ┌──────┐   ┌──────┐                                                     │
│ │ YES  │   │  NO  │  ◄── ERC1155 tokens (positionIds)                   │
│ │ idx=1│   │ idx=2│                                                     │
│ └──────┘   └──────┘                                                     │
│    │           │                                                        │
│    └─────┬─────┘                                                        │
│          ▼ mergePositions                                               │
│   COLLATERAL (USDC)                                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Contract Address

| Network | Address |
|---------|---------|
| Polygon Mainnet | `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045` |

---

## Key Concepts

### Condition

A condition represents a question with multiple possible outcomes. For Polymarket binary markets:
- **outcomeSlotCount:** 2 (YES and NO)
- **oracle:** UmaCtfAdapter address
- **questionId:** keccak256 hash of ancillaryData

### Condition ID

Unique identifier for each condition:

```solidity
bytes32 conditionId = keccak256(abi.encodePacked(
    address oracle,        // UmaCtfAdapter address
    bytes32 questionId,    // keccak256(ancillaryData)
    uint256 outcomeSlotCount  // 2 for binary
));

// Or via CTF function:
bytes32 conditionId = CTF.getConditionId(oracle, questionId, 2);
```

### Index Set

A 256-bit array indicating which outcome slots are included:

| Outcome | Index Set | Binary |
|---------|-----------|--------|
| YES (first) | 1 | 0b01 |
| NO (second) | 2 | 0b10 |
| Both | 3 | 0b11 |

### Collection ID

Identifies a specific outcome collection:

```solidity
bytes32 collectionId = CTF.getCollectionId(
    bytes32 parentCollectionId,  // 0x0 for root
    bytes32 conditionId,
    uint256 indexSet             // 1 for YES, 2 for NO
);
```

### Position ID (Token ID)

The ERC1155 token ID for a specific position:

```solidity
uint256 positionId = CTF.getPositionId(
    IERC20 collateralToken,  // USDC address
    bytes32 collectionId
);

// This is the ERC1155 tokenId used for transfers and balance queries
```

---

## Core Functions

### prepareCondition

Initializes a new condition (called by oracle/adapter).

**Source:** [ConditionalTokens.sol#L67-L83](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol#L67-L83)

```solidity
function prepareCondition(
    address oracle,           // UmaCtfAdapter address
    bytes32 questionId,       // keccak256(ancillaryData)
    uint outcomeSlotCount     // 2 for binary
) external;
```

**Events:**
```solidity
event ConditionPreparation(
    bytes32 indexed conditionId,
    address indexed oracle,
    bytes32 indexed questionId,
    uint outcomeSlotCount
);
```

**Requirements:**
- outcomeSlotCount must be 2-256
- Condition must not already exist

### splitPosition

Converts collateral into outcome tokens.

**Source:** [ConditionalTokens.sol#L105-L159](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol#L105-L159)

```solidity
function splitPosition(
    IERC20 collateralToken,        // USDC address
    bytes32 parentCollectionId,    // 0x0 for root positions
    bytes32 conditionId,
    uint256[] calldata partition,  // [1, 2] for binary split
    uint256 amount                 // Amount of collateral
) external;
```

**Effect:**
- Transfers `amount` USDC from caller to CTF contract
- Mints `amount` of each outcome token to caller

**Example:**
```solidity
// Split 100 USDC into 100 YES + 100 NO tokens
CTF.splitPosition(
    USDC,           // 0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174
    bytes32(0),     // root level
    conditionId,
    [1, 2],         // YES and NO
    100e6           // 100 USDC (6 decimals)
);
```

**Events:**
```solidity
event PositionSplit(
    address indexed stakeholder,
    IERC20 collateralToken,
    bytes32 indexed parentCollectionId,
    bytes32 indexed conditionId,
    uint256[] partition,
    uint256 amount
);
```

### mergePositions

Converts outcome tokens back into collateral.

**Source:** [ConditionalTokens.sol#L161-L208](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol#L161-L208)

```solidity
function mergePositions(
    IERC20 collateralToken,
    bytes32 parentCollectionId,
    bytes32 conditionId,
    uint256[] calldata partition,
    uint256 amount
) external;
```

**Effect:**
- Burns `amount` of each outcome token from caller
- Transfers `amount` USDC back to caller

**Requirement:** Caller must hold `amount` of ALL tokens in the partition

**Events:**
```solidity
event PositionsMerge(
    address indexed stakeholder,
    IERC20 collateralToken,
    bytes32 indexed parentCollectionId,
    bytes32 indexed conditionId,
    uint256[] partition,
    uint256 amount
);
```

### reportPayouts

Called by oracle to report condition outcome.

**Source:** [ConditionalTokens.sol#L80-L103](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol#L80-L103)

```solidity
function reportPayouts(
    bytes32 questionId,
    uint256[] calldata payouts  // Payout numerators
) external;
```

**Caller:** Must be the oracle address set in prepareCondition

**Payout Vectors:**

| Outcome | Payouts | Effect |
|---------|---------|--------|
| YES wins | `[1, 0]` | YES tokens redeem 1:1, NO tokens worthless |
| NO wins | `[0, 1]` | NO tokens redeem 1:1, YES tokens worthless |
| Invalid | `[1, 1]` | Both tokens redeem at 50% |

**Events:**
```solidity
event ConditionResolution(
    bytes32 indexed conditionId,
    address indexed oracle,
    bytes32 indexed questionId,
    uint outcomeSlotCount,
    uint256[] payoutNumerators
);
```

### redeemPositions

Converts winning tokens into collateral after resolution.

**Source:** [ConditionalTokens.sol#L210-L254](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol#L210-L254)

```solidity
function redeemPositions(
    IERC20 collateralToken,
    bytes32 parentCollectionId,
    bytes32 conditionId,
    uint256[] calldata indexSets  // Which positions to redeem
) external;
```

**Effect:**
- Burns outcome tokens from caller
- Transfers proportional collateral based on payout vector

**Example:**
```solidity
// Redeem 100 YES tokens after YES wins (payout [1, 0])
CTF.redeemPositions(
    USDC,
    bytes32(0),
    conditionId,
    [1]  // Only redeem YES tokens (indexSet=1)
);
// Receives 100 USDC

// Can also redeem both at once:
CTF.redeemPositions(
    USDC,
    bytes32(0),
    conditionId,
    [1, 2]  // Redeem both YES and NO
);
```

**Events:**
```solidity
event PayoutRedemption(
    address indexed redeemer,
    IERC20 indexed collateralToken,
    bytes32 indexed parentCollectionId,
    bytes32 conditionId,
    uint256[] indexSets,
    uint256 payout
);
```

---

## ID Calculation Examples

### Full Position ID Derivation

```solidity
// Given:
address oracle = 0x2F5e3684cb1F318ec51b00Edba38d79Ac2c0aA9d;  // UmaCtfAdapter
bytes32 questionId = keccak256("Will BTC hit 100k?");
address USDC = 0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174;

// Step 1: Calculate conditionId
bytes32 conditionId = keccak256(abi.encodePacked(
    oracle,
    questionId,
    uint256(2)
));

// Step 2: Calculate collectionIds
bytes32 yesCollectionId = CTF.getCollectionId(bytes32(0), conditionId, 1);
bytes32 noCollectionId = CTF.getCollectionId(bytes32(0), conditionId, 2);

// Step 3: Calculate positionIds (ERC1155 tokenIds)
uint256 yesTokenId = CTF.getPositionId(USDC, yesCollectionId);
uint256 noTokenId = CTF.getPositionId(USDC, noCollectionId);

// Now you can check balances:
uint256 yesBalance = CTF.balanceOf(user, yesTokenId);
uint256 noBalance = CTF.balanceOf(user, noTokenId);
```

### getCollectionId Implementation

**Source:** [CTHelpers.sol#L10-L28](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/CTHelpers.sol#L10-L28)

```solidity
function getCollectionId(
    bytes32 parentCollectionId,
    bytes32 conditionId,
    uint256 indexSet
) public pure returns (bytes32) {
    return bytes32(uint256(parentCollectionId) +
           uint256(keccak256(abi.encodePacked(conditionId, indexSet))));
}
```

### getPositionId Implementation

**Source:** [CTHelpers.sol#L30-L38](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/CTHelpers.sol#L30-L38)

```solidity
function getPositionId(
    IERC20 collateralToken,
    bytes32 collectionId
) public pure returns (uint256) {
    return uint256(keccak256(abi.encodePacked(collateralToken, collectionId)));
}
```

---

## ERC1155 Interface

The CTF contract implements ERC1155 for outcome tokens:

```solidity
// Check balance
function balanceOf(address account, uint256 id)
    external view returns (uint256);

// Batch balance check
function balanceOfBatch(address[] accounts, uint256[] ids)
    external view returns (uint256[] memory);

// Transfer tokens
function safeTransferFrom(
    address from,
    address to,
    uint256 id,
    uint256 amount,
    bytes data
) external;

// Approve operator
function setApprovalForAll(address operator, bool approved) external;

// Check approval
function isApprovedForAll(address account, address operator)
    external view returns (bool);
```

---

## Querying Condition State

### Check if Condition Exists

```solidity
// Returns outcomeSlotCount, 0 if not prepared
function getOutcomeSlotCount(bytes32 conditionId)
    external view returns (uint256);
```

### Check if Condition is Resolved

```solidity
// Returns payout denominator, 0 if not resolved
function payoutDenominator(bytes32 conditionId)
    external view returns (uint256);

// Returns payout numerator for specific outcome
function payoutNumerators(bytes32 conditionId, uint256 index)
    external view returns (uint256);
```

---

## Deep Positions (Advanced)

CTF supports nested conditions (deep positions):

```
Collateral
    └── Condition A (YES/NO)
            └── Condition B (YES/NO)
                    └── Position: A.YES & B.YES
                    └── Position: A.YES & B.NO
```

This allows complex conditional markets, but Polymarket primarily uses single-level binary markets.

---

## References

- [Gnosis CTF Contracts](https://github.com/gnosis/conditional-tokens-contracts)
- [CTF Documentation](https://conditional-tokens.readthedocs.io/)
- [Polymarket CTF Overview](https://docs.polymarket.com/developers/CTF/overview)
- [CTF Developer Guide](https://conditional-tokens.readthedocs.io/en/latest/developer-guide.html)
