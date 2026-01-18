# Polymarket Market Creation

> Contract-level walkthrough with real transaction examples

---

## Table of Contents

1. [Overview](#overview)
2. [Contract Architecture](#contract-architecture)
3. [Contracts Involved](#contracts-involved)
4. [Step 0: Pre-Approval (USDC)](#step-0-pre-approval-usdc)
5. [Step 1: Initialize Market](#step-1-initialize-market)
6. [Internal Functions](#internal-functions)
7. [CTF Integration](#ctf-integration-preparecondition)
8. [Events Emitted During Initialize](#events-emitted-during-initialize)
9. [Real Transaction: Initialize](#real-transaction-initialize)
10. [ID Derivation Reference](#id-derivation-reference)
11. [Post-Creation Flow](#post-creation-what-happens-next)
12. [Resolution: The resolve() Function](#resolution-the-resolve-function)
13. [Real Transaction: Resolve](#real-transaction-resolve)
14. [Additional Transaction Examples](#additional-transaction-examples)
15. [Event Comparison & Topic Hashes](#event-comparison-initialize-vs-resolve)
16. [How to Find More Transactions](#how-to-find-more-transactions)
17. [Complete Lifecycle Summary](#complete-lifecycle-summary)
18. [References](#references)

---

## Overview

This document provides an in-depth, contract-level analysis of how Polymarket markets are created, from the initial `initialize` call through to resolution. Each step includes:
- Solidity function signatures and source code
- Storage structures and state changes
- Events emitted
- Real transaction examples from PolygonScan

---

## Contract Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    MARKET CREATION FLOW                                    │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│   MARKET CREATOR                                                           │
│        │                                                                   │
│        ▼                                                                   │
│   ┌─────────────────┐                                                      │
│   │ 1. USDC.approve │ ──▶ Approve UmaCtfAdapter for reward amount          │
│   └────────┬────────┘                                                      │
│            │                                                               │
│            ▼                                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                      UmaCtfAdapter.initialize()                     │  │
│   │                                                                     │  │
│   │   2. Validate reward token (whitelist check)                        │  │
│   │   3. Append creator address to ancillaryData                        │  │
│   │   4. Generate questionId = keccak256(ancillaryData)                 │  │
│   │   5. _saveQuestion() → Store in questions mapping                   │  │
│   │   6. CTF.prepareCondition() → Create condition with 2 outcomes      │  │
│   │   7. _requestPrice() → Request price from OptimisticOracle          │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│            │                                                               │
│            ├──────────────────────┬────────────────────────┐               │
│            ▼                      ▼                        ▼               │
│   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐          │
│   │  CTF Contract   │   │  Optimistic     │   │    USDC.e       │          │
│   │                 │   │  Oracle V2      │   │   Transfer      │          │
│   │  prepareCondition│  │  requestPrice   │   │  (reward)       │          │
│   └─────────────────┘   └─────────────────┘   └─────────────────┘          │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Contracts Involved

| Contract | Address (Polygon) | Role | Source |
|----------|-------------------|------|--------|
| **UmaCtfAdapter V3** | `0x2F5e3684cb1F318ec51b00Edba38d79Ac2c0aA9d` | Bridge between UMA and CTF | [GitHub](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol) |
| **UmaCtfAdapter V2** | `0x6A9D222616C90FcA5754cd1333cFD9b7fb6a4F74` | Legacy adapter (still active) | [GitHub](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol) |
| **Conditional Tokens (CTF)** | `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045` | ERC1155 outcome tokens | [GitHub](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol) |
| **OptimisticOracleV2** | `0xee3afe347d5c74317041e2618c49534daf887c24` | UMA oracle | [GitHub](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol) |
| **USDC.e** | `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` | Collateral token | [PolygonScan](https://polygonscan.com/address/0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174) |

---

## Step 0: Pre-Approval (USDC)

Before calling `initialize`, the market creator must approve the UmaCtfAdapter to spend USDC for the proposer reward.

### Function

```solidity
// On USDC.e contract (0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174)
function approve(address spender, uint256 amount) external returns (bool);
```

### Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| `spender` | UmaCtfAdapter address | The adapter that will transfer reward |
| `amount` | reward amount (e.g., 2e6 = 2 USDC) | Amount to approve |

### Event Emitted

```solidity
event Approval(
    address indexed owner,    // Market creator
    address indexed spender,  // UmaCtfAdapter
    uint256 value            // Approved amount
);
```

---

## Step 1: Initialize Market

The main entry point for market creation.

> **Source:** [UmaCtfAdapter.sol#L103-L127](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L103-L127)

### Function Signature

```solidity
function initialize(
    bytes memory ancillaryData,
    address rewardToken,
    uint256 reward,
    uint256 proposalBond,
    uint256 liveness
) external returns (bytes32 questionID);
```

### Parameters Explained

| Parameter | Type | Description | Example Value |
|-----------|------|-------------|---------------|
| `ancillaryData` | bytes | UTF-8 encoded question + resolution rules | See format below |
| `rewardToken` | address | ERC20 token for proposer reward | `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` (USDC.e) |
| `reward` | uint256 | Amount paid to successful proposer | `2000000` (2 USDC) |
| `proposalBond` | uint256 | Bond required from proposer (0 = default ~750 USDC) | `500000000` (500 USDC) |
| `liveness` | uint256 | Dispute window in seconds (0 = default 7200) | `0` |

### Ancillary Data Format

```
q: Will [EVENT DESCRIPTION]?
res_data: p1: 0, p2: 1, p3: 0.5. Where p1 corresponds to No, p2 to Yes, p3 to Unknown/50-50.
[Additional resolution rules and criteria]
```

**Real Example from Transaction:**
```
q: Will Hull City AFC win on 2026-02-14?
res_data: p1: 0, p2: 1, p3: 0.5. Where p1 corresponds to No, p2 to Yes, p3 to Unknown/50-50.
Match: Hull City AFC vs Preston North End scheduled for 2026-02-14.
Resolution: YES if Hull City wins, NO if Hull City loses or draws.
```

---

## Internal Functions

### _saveQuestion

Stores the question metadata in contract storage.

> **Source:** [UmaCtfAdapter.sol#L272-L288](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L272-L288)

#### Storage Structure

> **Struct Definition:** [IUmaCtfAdapter.sol#L4-L24](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/interfaces/IUmaCtfAdapter.sol#L4-L24)

```solidity
struct QuestionData {
    uint256 requestTimestamp;           // When OO request was made
    uint256 reward;                     // Proposer incentive
    uint256 proposalBond;               // Required bond amount
    uint256 liveness;                   // Dispute window duration
    uint256 emergencyResolutionTimestamp; // Safety period expiry
    bool resolved;                      // Has been settled
    bool paused;                        // Temporarily frozen
    bool reset;                         // Has been disputed/reset
    bool refund;                        // Reward should be refunded
    address rewardToken;                // Payment token
    address creator;                    // Who initialized
    bytes ancillaryData;               // Question + rules
}

mapping(bytes32 => QuestionData) public questions;
```

#### Implementation Logic

```solidity
function _saveQuestion(
    bytes32 questionID,
    bytes memory ancillaryData,
    uint256 requestTimestamp,
    address rewardToken,
    uint256 reward,
    uint256 proposalBond,
    uint256 liveness
) internal {
    questions[questionID] = QuestionData({
        requestTimestamp: requestTimestamp,
        reward: reward,
        proposalBond: proposalBond,
        liveness: liveness,
        emergencyResolutionTimestamp: 0,
        resolved: false,
        paused: false,
        reset: false,
        refund: false,
        rewardToken: rewardToken,
        creator: msg.sender,
        ancillaryData: ancillaryData
    });
}
```

---

### _requestPrice

Requests a price from UMA's Optimistic Oracle.

> **Source:** [UmaCtfAdapter.sol#L290-L327](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L290-L327)

#### Implementation Flow

```solidity
function _requestPrice(
    uint256 requestTimestamp,
    bytes memory ancillaryData,
    address rewardToken,
    uint256 reward,
    uint256 proposalBond
) internal {
    // 1. Transfer reward from creator to adapter
    if (reward > 0) {
        IERC20(rewardToken).safeTransferFrom(msg.sender, address(this), reward);
    }

    // 2. Approve OptimisticOracle to spend reward
    IERC20(rewardToken).safeApprove(address(optimisticOracle), reward);

    // 3. Request price with YES_OR_NO_QUERY identifier
    optimisticOracle.requestPrice(
        YES_OR_NO_IDENTIFIER,     // bytes32 identifier
        requestTimestamp,          // uint256 timestamp
        ancillaryData,            // bytes ancillaryData
        IERC20(rewardToken),      // IERC20 currency
        reward                    // uint256 reward
    );

    // 4. Set custom bond if specified
    if (proposalBond > 0) {
        optimisticOracle.setBond(
            YES_OR_NO_IDENTIFIER,
            requestTimestamp,
            ancillaryData,
            proposalBond
        );
    }

    // 5. Configure as event-based request
    optimisticOracle.setEventBased(
        YES_OR_NO_IDENTIFIER,
        requestTimestamp,
        ancillaryData
    );

    // 6. Set callback recipient for disputes
    optimisticOracle.setCallbacks(
        YES_OR_NO_IDENTIFIER,
        requestTimestamp,
        ancillaryData,
        false,  // priceProposed callback
        true,   // priceDisputed callback
        false   // priceSettled callback
    );
}
```

---

## CTF Integration: prepareCondition

Creates the condition on the Conditional Tokens Framework.

> **Source:** [ConditionalTokens.sol#L67](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol#L67)

### CTF Contract Function

```solidity
function prepareCondition(
    address oracle,           // UmaCtfAdapter address
    bytes32 questionId,       // keccak256(ancillaryData)
    uint outcomeSlotCount     // 2 for binary markets
) external {
    // Validation
    require(outcomeSlotCount <= 256, "too many outcome slots");
    require(outcomeSlotCount > 1, "there should be more than one outcome slot");

    // Calculate conditionId
    bytes32 conditionId = CTHelpers.getConditionId(oracle, questionId, outcomeSlotCount);

    // Check not already prepared
    require(payoutNumerators[conditionId].length == 0, "condition already prepared");

    // Initialize payout vector
    payoutNumerators[conditionId] = new uint[](outcomeSlotCount);

    // Emit event
    emit ConditionPreparation(conditionId, oracle, questionId, outcomeSlotCount);
}
```

### ConditionId Calculation

> **Source:** [ConditionalTokens.sol#L232](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol#L232)

```solidity
function getConditionId(
    address oracle,
    bytes32 questionId,
    uint256 outcomeSlotCount
) internal pure returns (bytes32) {
    return keccak256(abi.encodePacked(oracle, questionId, outcomeSlotCount));
}
```

### Event Definition

> **Source:** [ConditionalTokens.sol#L12](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol#L12)

```solidity
event ConditionPreparation(
    bytes32 indexed conditionId,
    address indexed oracle,
    bytes32 indexed questionId,
    uint outcomeSlotCount
);
```

---

## Events Emitted During Initialize

### 1. Transfer (USDC.e)

```solidity
// Token transfer from creator to adapter
event Transfer(
    address indexed from,   // Market creator
    address indexed to,     // UmaCtfAdapter
    uint256 value          // Reward amount
);
```

### 2. Approval (USDC.e)

```solidity
// Adapter approves OptimisticOracle
event Approval(
    address indexed owner,    // UmaCtfAdapter
    address indexed spender,  // OptimisticOracle
    uint256 value            // Reward amount
);
```

### 3. RequestPrice (OptimisticOracleV2)

> **Source:** [OptimisticOracleV2.sol#L147](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol#L147)

```solidity
event RequestPrice(
    address indexed requester,  // UmaCtfAdapter
    bytes32 identifier,         // YES_OR_NO_QUERY
    uint256 timestamp,          // Request timestamp
    bytes ancillaryData,        // Question + rules
    address currency,           // USDC.e
    uint256 reward,            // Proposer reward
    uint256 finalFee           // UMA protocol fee
);
```

### 4. QuestionInitialized (UmaCtfAdapter)

> **Source:** [IUmaCtfAdapter.sol#L49-L56](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/interfaces/IUmaCtfAdapter.sol#L49-L56)

```solidity
event QuestionInitialized(
    bytes32 indexed questionID,
    uint256 indexed requestTimestamp,
    address indexed creator,
    bytes ancillaryData,
    address rewardToken,
    uint256 reward,
    uint256 proposalBond
);
```

---

## Real Transaction: Initialize

### Transaction Details

| Field | Value |
|-------|-------|
| **Hash** | [`0x2819c182be9373c061330a17f7b1e203cca699d59bb2f90f4487d95094a7ee75`](https://polygonscan.com/tx/0x2819c182be9373c061330a17f7b1e203cca699d59bb2f90f4487d95094a7ee75) |
| **Block** | 81796496 |
| **Timestamp** | 2026-01-18 05:09:24 UTC |
| **From** | `0x91430CaD2d3975766499717fA0D66A78D814E5c5` |
| **To** | `0x2F5e3684cb1F318ec51b00Edba38d79Ac2c0aA9d` (UmaCtfAdapter V3) |
| **Method** | `initialize(bytes,address,uint256,uint256,uint256)` |
| **Gas Used** | 1,042,140 / 6,000,000 (17.37%) |
| **Transaction Fee** | 0.109 POL (~$0.02) |

### Decoded Input Parameters

| Parameter | Value |
|-----------|-------|
| `ancillaryData` | `"q: Will Hull City AFC win on 2026-02-14?..."` |
| `rewardToken` | `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` (USDC.e) |
| `reward` | `2000000` (2 USDC) |
| `proposalBond` | `500000000` (500 USDC) |
| `liveness` | `0` (default 7200 seconds) |

### Events Emitted (In Order)

| # | Event | Contract | Key Data |
|---|-------|----------|----------|
| 749 | `Transfer` | USDC.e | 2 USDC: creator → adapter |
| 750 | `Approval` | USDC.e | Max allowance: creator → adapter |
| 753 | `RequestPrice` | OptimisticOracleV2 | identifier=YES_OR_NO_QUERY, reward=2M, finalFee=250M |
| 754 | `QuestionInitialized` | UmaCtfAdapter V3 | questionID, timestamp, creator |

### Generated IDs

```
questionId:       0x0b9d82e46047436f2e88c3fbca95d60cefb8ef0e5aae8f254be7806560c5e237
requestTimestamp: 1768712964
```

---

## ID Derivation Reference

### QuestionId

```solidity
// Calculated in initialize()
bytes memory data = abi.encodePacked(ancillaryData, msg.sender);
bytes32 questionID = keccak256(data);
```

### ConditionId

```solidity
// Calculated in CTF.prepareCondition()
bytes32 conditionId = keccak256(abi.encodePacked(
    address(UmaCtfAdapter),  // oracle
    questionID,              // from above
    uint256(2)              // outcomeSlotCount
));
```

### Position IDs (Token IDs)

> **Source:** [ConditionalTokens.sol#L240](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol#L240) and [#L248](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol#L248)

```solidity
// YES token
bytes32 yesCollectionId = CTF.getCollectionId(bytes32(0), conditionId, 1);
uint256 yesTokenId = CTF.getPositionId(USDC, yesCollectionId);

// NO token
bytes32 noCollectionId = CTF.getCollectionId(bytes32(0), conditionId, 2);
uint256 noTokenId = CTF.getPositionId(USDC, noCollectionId);
```

---

## Post-Creation: What Happens Next

After initialization, the market enters the **proposal phase**:

```
┌─────────────────────────────────────────────────────────────┐
│                    POST-INITIALIZATION                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   MARKET STATE: Awaiting Proposal                           │
│                                                             │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│   │  Proposer   │───▶│  Liveness   │───▶│   Settle    │     │
│   │  submits    │    │  (2 hours)  │    │   & Resolve │     │
│   │  answer     │    │             │    │             │     │
│   └─────────────┘    └──────┬──────┘    └─────────────┘     │
│                             │                               │
│                        ┌────▼────┐                          │
│                        │ Dispute │                          │
│                        │ (reset) │                          │
│                        └─────────┘                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### State Checks

```solidity
// Check if question is initialized
function isInitialized(bytes32 questionID) public view returns (bool);

// Check if ready to resolve (price available)
function ready(bytes32 questionID) public view returns (bool);

// Get expected payouts before resolution
function getExpectedPayouts(bytes32 questionID) public view returns (uint256[] memory);
```

---

## Token Minting & Initial Pricing

### IMPORTANT: No Tokens Minted During Initialize

**What `initialize()` creates:**
- ✅ `questionId` (identifier for the question)
- ✅ `conditionId` (identifier for the CTF condition)
- ✅ Oracle price request
- ✅ Metadata storage
- ❌ **NO TOKENS MINTED**
- ❌ **NO PRICES SET**

### When Are Tokens Actually Minted?

Tokens are minted **later** when traders want to participate:

#### Method 1: Direct Split (Manual Minting)

```solidity
// Trader manually splits USDC into outcome tokens
CTF.splitPosition(
    USDC,               // Collateral token
    bytes32(0),         // Parent collection (0 for root)
    conditionId,        // From initialization
    [1, 2],            // Partition: [YES, NO]
    100e6              // 100 USDC
)

Effect:
├── Burns 100 USDC from trader
├── Mints 100 YES tokens → trader
└── Mints 100 NO tokens → trader
```

#### Method 2: Exchange MINT Mode (Automatic)

When complementary orders are matched:

```
Alice wants: 100 YES @ $0.60
Bob wants:   100 NO  @ $0.40
Total: $0.60 + $0.40 = $1.00 ✓

Exchange automatically:
1. Takes 60 USDC from Alice
2. Takes 40 USDC from Bob
3. Calls CTF.splitPosition(100 USDC)
4. Mints 100 YES → gives to Alice
5. Mints 100 NO → gives to Bob
```

### Initial Price: NOT Hardcoded

**There is no on-chain initial price.**

Prices come from the orderbook:
- Market makers post initial orders (off-chain via CLOB)
- First orders often near $0.50/$0.50 (50% probability)
- But can start anywhere: $0.70/$0.30, $0.90/$0.10, etc.
- Determined by market makers, not smart contracts

**Example initial orderbook:**

```
Market just created (no tokens exist yet):

Market Maker posts first orders:
├── Sell YES @ $0.52 for 10,000 shares
├── Buy YES @ $0.48 for 10,000 shares
├── Sell NO @ $0.52 for 10,000 shares
└── Buy NO @ $0.48 for 10,000 shares

When first trader buys:
1. Matches with market maker's sell order
2. Exchange calls CTF.splitPosition()
3. Tokens minted for the FIRST TIME
4. Trade settles at ~$0.50 (market price)
```

### What IS Saved After Initialize?

**In UmaCtfAdapter:**
```solidity
questions[questionID] = QuestionData({
    requestTimestamp: 1704758400,
    reward: 50e6,                  // 50 USDC reward
    proposalBond: 750e6,           // 750 USDC bond
    liveness: 7200,                // 2 hours
    emergencyResolutionTimestamp: 0,
    resolved: false,
    paused: false,
    reset: false,
    refund: false,
    rewardToken: USDC,
    creator: msg.sender,
    ancillaryData: "q: Will X happen..."
});
```

**In CTF:**
```solidity
payoutNumerators[conditionId] = [0, 0]  // Empty until resolution
```

**In OptimisticOracle:**
```solidity
Price request with:
├── identifier: YES_OR_NO_QUERY
├── timestamp: 1704758400
├── ancillaryData: question text
├── currency: USDC
├── reward: 50 USDC
└── bond: 750 USDC
```

**NOT saved:**
- ❌ Token balances (no tokens exist)
- ❌ Prices (determined by orderbook)
- ❌ Liquidity (market makers provide later)

---

## Resolution: The resolve() Function

### Resolution Flow: UMA → CTF

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RESOLUTION FLOW: UMA → CTF                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. ANYONE calls UmaCtfAdapter.resolve(questionID)                  │
│     └─ Permissionless function                                      │
│                                                                     │
│  2. resolve() calls _resolve() internally                           │
│                                                                     │
│  3. _resolve() fetches price from UMA                               │
│     └─ optimisticOracle.settleAndGetPrice()                         │
│     └─ Returns: int256 price (1e18=YES, 0=NO, 0.5e18=UNKNOWN)       │
│                                                                     │
│  4. _resolve() converts price to payout vector                      │
│     └─ _constructPayouts(price)                                     │
│     └─ Returns: [1,0] for YES, [0,1] for NO, [1,1] for UNKNOWN      │
│                                                                     │
│  5. _resolve() reports to CTF                                       │
│     └─ ctf.reportPayouts(questionID, payouts)                       │
│                                                                     │
│  6. CTF stores the result                                           │
│     └─ payoutNumerators[conditionId] = payouts                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Who can call resolve():** Anyone (permissionless)
**What they send:** Nothing - they just trigger the fetch from UMA
**What they get:** Nothing - no reward for calling resolve()

> **Source:** [UmaCtfAdapter.sol#L169-L181](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L169-L181)

### Function Signature

```solidity
function resolve(bytes32 questionID) external;
```

### Implementation

```solidity
function resolve(bytes32 questionID) external {
    QuestionData storage questionData = questions[questionID];

    // Validation checks
    if (!_isInitialized(questionData)) revert NotInitialized();
    if (questionData.paused) revert Paused();
    if (questionData.resolved) revert Resolved();
    if (!_hasPrice(questionData)) revert NotReadyToResolve();

    return _resolve(questionID, questionData);
}

// Source: UmaCtfAdapter.sol#L329-L354
function _resolve(bytes32 questionID, QuestionData storage questionData) internal {
    // 1. Get price from OptimisticOracle
    int256 price = optimisticOracle.settleAndGetPrice(
        YES_OR_NO_IDENTIFIER,
        questionData.requestTimestamp,
        questionData.ancillaryData
    );

    // 2. Handle "ignore" price (reset if needed)
    if (price == _ignorePrice()) {
        return _reset(address(this), questionID, true, questionData);
    }

    // 3. Mark as resolved
    questionData.resolved = true;

    // 4. Refund reward if flagged
    if (questionData.refund) _refund(questionData);

    // 5. Construct payout vector
    uint256[] memory payouts = _constructPayouts(price);

    // 6. Report to CTF
    ctf.reportPayouts(questionID, payouts);

    // 7. Emit event
    emit QuestionResolved(questionID, price, payouts);
}
```

### Payout Construction

> **Source:** [UmaCtfAdapter.sol#L372-L395](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L372-L395)

```solidity
function _constructPayouts(int256 price) internal pure returns (uint256[] memory) {
    uint256[] memory payouts = new uint256[](2);

    // Validate price
    if (price != 0 && price != 0.5 ether && price != 1 ether) {
        revert InvalidOOPrice();
    }

    if (price == 0) {
        // NO wins
        payouts[0] = 0;
        payouts[1] = 1;
    } else if (price == 0.5 ether) {
        // Invalid/Unknown - 50/50 split
        payouts[0] = 1;
        payouts[1] = 1;
    } else {
        // YES wins (price == 1 ether)
        payouts[0] = 1;
        payouts[1] = 0;
    }

    return payouts;
}
```

---

## Dispute Mechanics

### Two-Tier Dispute System

Polymarket uses a two-tier dispute system to balance speed with accuracy:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DISPUTE FLOW                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  FIRST DISPUTE: Market Reset                                        │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Initial Proposal → Dispute! → Market RESETS                 │    │
│  │ • Burns 375 USDC (50% of proposer's bond)                   │    │
│  │ • Creates new price request                                 │    │
│  │ • Waiting for new proposal                                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  SECOND DISPUTE: DVM Escalation                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Reset Proposal → Dispute! → DVM Voting (48-96 hours)        │    │
│  │ • Burns another 375 USDC                                    │    │
│  │ • NO RESET this time                                        │    │
│  │ • UMA token holders vote on final answer                    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Bond Burning: Why 50%?

**Code Evidence:** [OptimisticOracleV2.sol - _computeBurnedBond()](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol)

```solidity
function _computeBurnedBond(Request storage request) private view returns (uint256) {
    return request.requestSettings.bond.div(2);  // 750 / 2 = 375 USDC
}
```

**Why burn bonds?** To prevent Sybil attacks:

```
Without burning:
├─ Attacker controls Wallet A (proposer) and Wallet B (disputer)
├─ A proposes wrong answer: 750 USDC
├─ B disputes: 750 USDC
├─ DVM decides A wins
├─ A gets: 750 + 750 + 55 = 1,555 USDC
└─ Net cost: 1,500 - 1,555 = -55 USDC (PROFIT!)

With 50% burning:
├─ A proposes: 750 USDC
├─ B disputes: 750 USDC
├─ Burned immediately: 375 USDC → UMA Store
├─ DVM decides A wins
├─ A gets: 750 + 375 + 55 = 1,180 USDC
└─ Net cost: 1,500 - 1,180 = 320 USDC LOSS ✓
```

### Payout Distribution After Dispute

| Outcome | Winner Receives | Loser Loses | UMA Store |
|---------|----------------|-------------|-----------|
| **Proposer Wins** | 1,180 USDC (bond + finalFee + reward + unburnedBond) | 755 USDC | 380 USDC |
| **Disputer Wins** | 1,130 USDC (bond + finalFee + unburnedBond) | 755 USDC | 380 USDC |

### DVM Vote Determines Winner (Binary Check)

**Critical Insight:** The system only checks if DVM agrees with the proposer—disputers don't submit alternative answers.

```solidity
// In _settle()
bool disputeSuccess = request.resolvedPrice != request.proposedPrice;

if (disputeSuccess) {
    // DVM disagreed with proposer → DISPUTER WINS
    recipient = request.disputer;
} else {
    // DVM agreed with proposer → PROPOSER WINS
    recipient = request.proposer;
}
```

This means if proposer says YES, disputer challenges, and DVM votes UNKNOWN—the **disputer wins** (because UNKNOWN ≠ YES).

See [resolution-qa.md](resolution-qa.md#dispute-mechanics) for comprehensive dispute scenarios.

---

## Real Transaction: Resolve

### Transaction Details

| Field | Value |
|-------|-------|
| **Hash** | [`0x1b80c5cf5fddfe0e67665d8c7f2383e38cbc40a460ee6d4180eac31c725fa891`](https://polygonscan.com/tx/0x1b80c5cf5fddfe0e67665d8c7f2383e38cbc40a460ee6d4180eac31c725fa891) |
| **Block** | 68931840 |
| **Timestamp** | 2025-03-11 |
| **From** | `0x39Ba731c...` |
| **To** | `0x6A9D222616C90FcA5754cd1333cFD9b7fb6a4F74` (UmaCtfAdapter V2) |
| **Method** | `resolve(bytes32)` |
| **Gas Used** | 181,046 / 1,000,000 (18.1%) |
| **Transaction Fee** | ~0.0081 POL ($0.001) |

### Decoded Input

```
questionID: 0xbc2cabceb51b13b951723795158c11edabdbcc77f2bb839a75e65cfe0a8f2b4b
```

### Events Emitted

| # | Event | Contract | Key Data |
|---|-------|----------|----------|
| 1 | `ConditionResolution` | CTF | conditionId=`0xf62dad...`, payouts=[1,0] |
| 2 | `QuestionResolved` | UmaCtfAdapter | price=1e18, payouts=[1,0] |
| 3 | `LogFeeTransfer` | POL Token | Gas fee distribution |

### Resolution Outcome

```
Settled Price: 1000000000000000000 (1.0 = YES)
Payout Vector: [1, 0]
Result:        YES wins - YES tokens redeem at 1 USDC, NO tokens at 0
```

---

## Additional Transaction Examples

### More Resolve Transactions

| Hash | Date | Outcome | Contract |
|------|------|---------|----------|
| [`0x08939e4d...`](https://polygonscan.com/tx/0x08939e4d18e3f51d2625a3cb6b1a5144ceb49b1c7a1c53731289851cdbe67f84) | Mar 2025 | - | V2 |
| [`0xb527af4e...`](https://polygonscan.com/tx/0xb527af4e73f8b346f14ce90c87ec42e96a51ceeaf3eb83f8820d51b940f2eca9) | Mar 2025 | - | V2 |
| [`0x1ee692de...`](https://polygonscan.com/tx/0x1ee692de383cda12b2c89cb1264bac7734283bfbef6045920004e8e7535e6244) | Mar 2025 | - | V2 |
| [`0x4ae264ea...`](https://polygonscan.com/tx/0x4ae264ea3178f490d1a58527d9773c9966dc4df61570604f8717edc33c8ad614) | Mar 2025 | - | V2 |

---

## Event Comparison: Initialize vs Resolve

| Aspect | Initialize | Resolve |
|--------|------------|---------|
| **Primary Contract** | UmaCtfAdapter | UmaCtfAdapter |
| **Secondary Calls** | CTF, OptimisticOracle | CTF, OptimisticOracle |
| **Main Event** | `QuestionInitialized` | `QuestionResolved` |
| **CTF Event** | `ConditionPreparation` | `ConditionResolution` |
| **OO Event** | `RequestPrice` | `Settle` |
| **Token Transfer** | USDC (reward) | None |
| **Avg Gas** | ~1,000,000 | ~180,000 |

### Event Topic Hashes

For filtering events on-chain:

| Event | Topic0 Hash |
|-------|-------------|
| `Transfer` (ERC20) | `0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef` |
| `Approval` (ERC20) | `0x8c5be1e5ebec7d5bd14f71427d1e84f3dd0314c0f7b2291e5b200ac8c7c3b925` |
| `RequestPrice` | `0xf1679315ff325c257a944e0ca1bfe7b26616039e9511f9610d4ba3eca851027b` |
| `QuestionInitialized` | `0xeee0897acd6893adcaf2ba5158191b3601098ab6bece35c5d57874340b64c5b7` |

---

## How to Find More Transactions

### Via PolygonScan UI

1. Go to adapter contract page
2. Click "Transactions" tab
3. Filter by method name ("Initialize" or "Resolve")

### Via PolygonScan API

```bash
# Get transactions for UmaCtfAdapter V3
curl "https://api.polygonscan.com/api?module=account&action=txlist&address=0x2F5e3684cb1F318ec51b00Edba38d79Ac2c0aA9d&sort=desc"
```

### Via Event Logs

```bash
# Get QuestionInitialized events
curl "https://api.polygonscan.com/api?module=logs&action=getLogs&address=0x2F5e3684cb1F318ec51b00Edba38d79Ac2c0aA9d&topic0=0xeee0897acd6893adcaf2ba5158191b3601098ab6bece35c5d57874340b64c5b7"
```

---

## Complete Lifecycle Summary

| Step | Function | Contract | Key Event | Source |
|------|----------|----------|-----------|--------|
| 0 | `approve()` | USDC.e | `Approval` | ERC20 |
| 1 | `initialize()` | UmaCtfAdapter | `QuestionInitialized` | [L103](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L103) |
| 1a | `prepareCondition()` | CTF | `ConditionPreparation` | [L67](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol#L67) |
| 1b | `requestPrice()` | OptimisticOracle | `RequestPrice` | [L104](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol#L104) |
| 2 | `proposePrice()` | OptimisticOracle | `ProposePrice` | [L242](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol#L242) |
| 3 | *(wait liveness)* | - | - | - |
| 4 | `resolve()` | UmaCtfAdapter | `QuestionResolved` | [L169](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L169) |
| 4a | `settleAndGetPrice()` | OptimisticOracle | `Settle` | [L318](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol#L318) |
| 4b | `reportPayouts()` | CTF | `ConditionResolution` | [L80](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol#L80) |
| 5 | `redeemPositions()` | CTF | `PayoutRedemption` | [L193](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol#L193) |

---

## References

### Internal Documentation
- [Resolution Q&A](resolution-qa.md) - Comprehensive Q&A on resolution flow, disputes, and edge cases
- [Market Lifecycle](market-lifecycle.md) - High-level overview of market phases
- [UMA Oracle](uma-oracle.md) - Detailed analysis of UMA's Optimistic Oracle

### Source Code (GitHub)

| Contract | Repository | Main File |
|----------|------------|-----------|
| **UmaCtfAdapter** | [Polymarket/uma-ctf-adapter](https://github.com/Polymarket/uma-ctf-adapter) | [UmaCtfAdapter.sol](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol) |
| **IUmaCtfAdapter** | [Polymarket/uma-ctf-adapter](https://github.com/Polymarket/uma-ctf-adapter) | [IUmaCtfAdapter.sol](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/interfaces/IUmaCtfAdapter.sol) |
| **ConditionalTokens** | [gnosis/conditional-tokens-contracts](https://github.com/gnosis/conditional-tokens-contracts) | [ConditionalTokens.sol](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol) |
| **OptimisticOracleV2** | [UMAprotocol/protocol](https://github.com/UMAprotocol/protocol) | [OptimisticOracleV2.sol](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol) |

### Documentation

- [Polymarket Resolution Docs](https://docs.polymarket.com/developers/resolution/UMA)
- [UMA Optimistic Oracle Docs](https://docs.uma.xyz/developers/optimistic-oracle)
- [Gnosis CTF Developer Guide](https://conditional-tokens.readthedocs.io/en/latest/developer-guide.html)

### Block Explorers (PolygonScan)

| Contract | Link |
|----------|------|
| UmaCtfAdapter V3 | [View](https://polygonscan.com/address/0x2F5e3684cb1F318ec51b00Edba38d79Ac2c0aA9d) |
| UmaCtfAdapter V2 | [View](https://polygonscan.com/address/0x6A9D222616C90FcA5754cd1333cFD9b7fb6a4F74) |
| CTF | [View](https://polygonscan.com/address/0x4D97DCd97eC945f40cF65F87097ACe5EA0476045) |
| OptimisticOracleV2 | [View](https://polygonscan.com/address/0xee3afe347d5c74317041e2618c49534daf887c24) |
| USDC.e | [View](https://polygonscan.com/address/0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174) |
