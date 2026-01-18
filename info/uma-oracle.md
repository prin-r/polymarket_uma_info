# UMA Optimistic Oracle Deep Dive

> Contract-level analysis of how UMA's Optimistic Oracle works

**Last Updated:** 2026-01-18

---

## Table of Contents

1. [Overview](#overview)
2. [Oracle Version Comparison](#oracle-version-comparison)
3. [Contract Architecture](#contract-architecture)
4. [Core Data Structures](#core-data-structures)
5. [Request Lifecycle Flow](#request-lifecycle-flow)
6. [Core Functions (Contract Level)](#core-functions-contract-level)
7. [Bond Mechanics & Economics](#bond-mechanics--economics)
8. [DVM Escalation & Voting](#dvm-escalation--voting)
9. [Production Transaction Examples](#production-transaction-examples)
10. [Contract Addresses](#contract-addresses)
11. [References](#references)

---

## Overview

UMA's Optimistic Oracle is an **escalation-game-based oracle system** where:
1. Anyone can propose an answer with a bond
2. Anyone can dispute during the liveness period
3. Only disputed answers go to full arbitration (DVM voting)

The key insight: **99%+ of requests settle optimistically** without needing the expensive DVM voting process.

```
┌───────────────────────────────────────────────────────────────────────┐
│                    UMA OPTIMISTIC ORACLE FLOW                         │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────────────┐   │
│   │ Request │───▶│ Propose │───▶│Liveness │───▶│ Settle (Accept) │   │
│   │  Price  │    │  Price  │    │  (2hr)  │    │                 │   │
│   └─────────┘    └────┬────┘    └────┬────┘    └─────────────────┘   │
│                       │              │                                │
│                       │         ┌────▼────┐                          │
│                       │         │ Dispute │                          │
│                       │         └────┬────┘                          │
│                       │              │                                │
│              ┌────────▼──────────────▼────────┐                      │
│              │   First Dispute = Reset        │                      │
│              │   Second Dispute = DVM Vote    │                      │
│              └────────────────────────────────┘                      │
│                              │                                        │
│                     ┌────────▼────────┐                              │
│                     │    DVM Vote     │                              │
│                     │   (48-96 hrs)   │                              │
│                     └─────────────────┘                              │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Oracle Version Comparison

UMA has released multiple versions of the Optimistic Oracle. Here's how they differ:

### Version Summary

| Feature | OOv1 (Deprecated) | OOv2 (Active) | OOv3 (Active) |
|---------|-------------------|---------------|---------------|
| **Status** | Deprecated | Production | Production |
| **Request Model** | Request-Propose | Request-Propose | Assertion-based |
| **Who Proposes** | Third-party proposers | Third-party proposers | Asserter (self) |
| **Integration Complexity** | Moderate | Moderate | Simpler |
| **Callbacks** | Limited | Full (propose/dispute/settle) | Simplified |
| **Custom Liveness** | No | Yes | Yes |
| **Event-based Requests** | No | Yes | Built-in |

### Which Version to Use?

**Use OOv2 when:**
- You need **third-party proposers** to answer data requests
- Building prediction markets, sports betting, insurance protocols
- You want the market to incentivize external participants
- **Polymarket uses OOv2**

**Use OOv3 when:**
- Your protocol can **assert its own data** and needs verification
- Building cross-chain bridges, content moderation, transaction verification
- Simpler integration is preferred
- You don't need external proposers

### Key Differences Explained

**OOv2: Request-Propose Model**
```
Contract → requestPrice() → Waits for proposer → proposePrice() → Liveness → settle()
                                    ↑
                            Third-party proposer
```

**OOv3: Assertion Model**
```
Contract → assertTruth(claim, bond) → Liveness → settleAssertion()
                ↑
        Asserter proposes own data
```

---

## Contract Architecture

### GitHub Repository Structure

**Main Repository:** [UMAprotocol/protocol](https://github.com/UMAprotocol/protocol)

```
packages/core/contracts/
├── optimistic-oracle-v2/
│   ├── implementation/
│   │   └── OptimisticOracleV2.sol          # Core OOv2 contract
│   └── interfaces/
│       ├── OptimisticOracleV2Interface.sol # Interface definitions
│       └── SkinnyOptimisticOracleV2Interface.sol
├── optimistic-oracle-v3/
│   └── implementation/
│       └── OptimisticOracleV3.sol          # Core OOv3 contract
└── data-verification-mechanism/
    └── implementation/
        └── VotingV2.sol                     # DVM voting contract
```

### Key Contract Links

| Contract | GitHub Link |
|----------|-------------|
| **OptimisticOracleV2.sol** | [View Source](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol) |
| **OptimisticOracleV2Interface.sol** | [View Source](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/interfaces/OptimisticOracleV2Interface.sol) |
| **VotingV2.sol** | [View Source](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/data-verification-mechanism/implementation/VotingV2.sol) |
| **OptimisticOracleV3.sol** | [View Source](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v3/implementation/OptimisticOracleV3.sol) |

### Contract Inheritance

```solidity
OptimisticOracleV2
    ├── implements OptimisticOracleV2Interface
    ├── uses FinderInterface (for DVM, Store, IdentifierWhitelist)
    ├── uses IERC20 (for bond/reward tokens)
    └── inherits ReentrancyGuard
```

---

## Core Data Structures

### State Enum

The contract tracks request lifecycle through these states:

```solidity
// From OptimisticOracleV2Interface.sol
enum State {
    Invalid,      // 0 - Request doesn't exist
    Requested,    // 1 - Price requested, awaiting proposal
    Proposed,     // 2 - Price proposed, in liveness period
    Expired,      // 3 - Liveness passed, no dispute (can settle)
    Disputed,     // 4 - Proposal disputed, awaiting DVM
    Resolved,     // 5 - DVM returned price (can settle)
    Settled       // 6 - Final price set, payouts distributed
}
```

**State Transition Diagram:**
```
                    ┌──────────────────────────────────────────┐
                    │                                          │
                    ▼                                          │
Invalid ──► Requested ──► Proposed ──┬──► Expired ──► Settled │
                                     │                    ▲    │
                                     │                    │    │
                                     └──► Disputed ──► Resolved
```

### Request Struct

```solidity
// From OptimisticOracleV2Interface.sol
struct Request {
    address proposer;                    // Address that proposed the price
    address disputer;                    // Address that disputed (0x0 if none)
    IERC20 currency;                     // ERC20 token for bonds/rewards
    bool settled;                        // Whether request has been settled
    RequestSettings requestSettings;     // Custom configuration
    int256 proposedPrice;                // The proposed price value
    int256 resolvedPrice;                // Final price (from expiry or DVM)
    uint256 expirationTime;              // When liveness period ends
    uint256 reward;                      // Reward for successful proposer
    uint256 finalFee;                    // Fee paid to DVM/Store
}
```

### RequestSettings Struct

```solidity
struct RequestSettings {
    bool eventBased;                     // If true, timestamp is ignored
    bool refundOnDispute;                // Refund reward to requester on dispute
    bool callbackOnPriceProposed;        // Call requester.priceProposed()
    bool callbackOnPriceDisputed;        // Call requester.priceDisputed()
    bool callbackOnPriceSettled;         // Call requester.priceSettled()
    uint256 bond;                        // Custom bond amount (0 = default)
    uint256 customLiveness;              // Custom liveness period (0 = default)
}
```

### Request ID Calculation

Requests are identified by a unique hash:

```solidity
bytes32 requestId = keccak256(
    abi.encodePacked(
        requester,      // Address that requested the price
        identifier,     // Price identifier (e.g., "YES_OR_NO_QUERY")
        timestamp,      // Request timestamp
        ancillaryData   // Question/context data
    )
);
```

---

## Request Lifecycle Flow

### Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        COMPLETE OOv2 LIFECYCLE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. REQUEST PHASE                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Requester calls requestPrice()                                       │   │
│  │ • Transfers reward to OOv2 contract                                  │   │
│  │ • State: Invalid → Requested                                         │   │
│  │ • Emits RequestPrice event                                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  2. PROPOSAL PHASE                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Proposer calls proposePrice()                                        │   │
│  │ • Transfers bond + finalFee to OOv2 contract                         │   │
│  │ • Sets expirationTime = now + liveness                               │   │
│  │ • State: Requested → Proposed                                        │   │
│  │ • Emits ProposePrice event                                           │   │
│  │ • Optional: Callback to requester.priceProposed()                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                    ┌───────────────┴───────────────┐                       │
│                    ▼                               ▼                        │
│  3a. NO DISPUTE (Happy Path)          3b. DISPUTE PATH                     │
│  ┌────────────────────────────┐      ┌─────────────────────────────────┐   │
│  │ Liveness period passes     │      │ Disputer calls disputePrice()   │   │
│  │ • No one disputes          │      │ • Transfers bond + finalFee     │   │
│  │ • State: Proposed → Expired│      │ • Escalates to DVM              │   │
│  └────────────────────────────┘      │ • State: Proposed → Disputed    │   │
│                    │                  │ • Emits DisputePrice event      │   │
│                    │                  └─────────────────────────────────┘   │
│                    │                               │                        │
│                    │                               ▼                        │
│                    │                  ┌─────────────────────────────────┐   │
│                    │                  │ DVM Voting (48-96 hours)        │   │
│                    │                  │ • Commit phase (24-48h)         │   │
│                    │                  │ • Reveal phase (24-48h)         │   │
│                    │                  │ • State: Disputed → Resolved    │   │
│                    │                  └─────────────────────────────────┘   │
│                    │                               │                        │
│                    └───────────────┬───────────────┘                       │
│                                    ▼                                        │
│  4. SETTLEMENT PHASE                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Anyone calls settle()                                                │   │
│  │ • Distributes bonds based on outcome                                 │   │
│  │ • Winner gets: bond + reward + loser's bond share                    │   │
│  │ • State: Expired/Resolved → Settled                                  │   │
│  │ • Emits Settle event                                                 │   │
│  │ • Optional: Callback to requester.priceSettled()                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Core Functions (Contract Level)

### 1. requestPrice()

**Purpose:** Initiate a price request

**GitHub:** [OptimisticOracleV2.sol#L97-L167](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol)

```solidity
function requestPrice(
    bytes32 identifier,         // Price identifier (e.g., "YES_OR_NO_QUERY")
    uint256 timestamp,          // Request timestamp (ignored if eventBased)
    bytes memory ancillaryData, // Question text + resolution rules
    IERC20 currency,            // Bond/reward currency (e.g., USDC)
    uint256 reward              // Reward offered to proposer
) external returns (uint256 totalBond);
```

**What happens internally:**
1. Validates identifier is whitelisted
2. Validates ancillary data length ≤ OO limit
3. Creates unique request ID from hash
4. Initializes Request struct with state = Requested
5. Transfers reward from requester to contract
6. Returns initial bond requirement (2x finalFee)

**Example call:**
```solidity
// UmaCtfAdapter calling OOv2
optimisticOracle.requestPrice(
    bytes32("YES_OR_NO_QUERY"),
    block.timestamp,
    abi.encodePacked("q: Will ETH reach $5000 by Dec 31? p1: Yes, p2: No"),
    IERC20(USDC),
    5e6  // 5 USDC reward
);
```

---

### 2. proposePrice()

**Purpose:** Submit a proposed answer with bond

**GitHub:** [OptimisticOracleV2.sol#L169-L240](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol)

```solidity
function proposePrice(
    address requester,          // Address that made the request
    bytes32 identifier,         // Must match original request
    uint256 timestamp,          // Must match original request
    bytes memory ancillaryData, // Must match original request
    int256 proposedPrice        // The proposed value
) external returns (uint256 totalBond);
```

**What happens internally:**
1. Validates state == Requested
2. Sets request.proposer = msg.sender
3. Sets request.proposedPrice = proposedPrice
4. Calculates bond = customBond || defaultBond
5. Calculates expirationTime = now + customLiveness || defaultLiveness
6. Transfers (bond + finalFee) from proposer to contract
7. State transition: Requested → Proposed
8. Emits ProposePrice event
9. Optional: Calls requester.priceProposed() callback

**Price values for YES_OR_NO_QUERY:**
```solidity
int256 YES = 1e18;      // 1000000000000000000
int256 NO = 0;          // 0
int256 UNKNOWN = 5e17;  // 500000000000000000 (0.5)
```

---

### 3. disputePrice()

**Purpose:** Challenge a proposal and escalate to DVM

**GitHub:** [OptimisticOracleV2.sol#L242-L340](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol)

```solidity
function disputePrice(
    address requester,
    bytes32 identifier,
    uint256 timestamp,
    bytes memory ancillaryData
) external returns (uint256 totalBond);
```

**What happens internally:**
1. Validates state == Proposed
2. Validates block.timestamp < expirationTime (within liveness)
3. Sets request.disputer = msg.sender
4. Transfers (bond + finalFee) from disputer to contract
5. State transition: Proposed → Disputed
6. **Calls DVM:** `oracle.requestPrice(identifier, timestamp, ancillaryData)`
7. Burns half of loser's bond (sent to Store)
8. Emits DisputePrice event
9. Optional: Calls requester.priceDisputed() callback
10. Optional: Refunds reward if refundOnDispute is set

**Key insight:** The dispute triggers the DVM request:
```solidity
// Inside disputePrice()
_getOracle().requestPrice(identifier, timestamp, ancillaryData);
```

---

### 4. settle()

**Purpose:** Finalize request and distribute bonds

**GitHub:** [OptimisticOracleV2.sol#L342-L450](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol)

```solidity
function settle(
    address requester,
    bytes32 identifier,
    uint256 timestamp,
    bytes memory ancillaryData
) external returns (uint256 payout);
```

**Settlement scenarios:**

**Scenario A: Expired (No Dispute)**
```solidity
if (state == State.Expired) {
    // Proposer was correct (no one challenged)
    resolvedPrice = proposedPrice;
    payout = bond + finalFee + reward;
    // Transfer payout to proposer
}
```

**Scenario B: Resolved (DVM Decided)**
```solidity
if (state == State.Resolved) {
    resolvedPrice = _getOracle().getPrice(identifier, timestamp, ancillaryData);

    if (resolvedPrice == proposedPrice) {
        // Proposer was correct
        payout = proposerBond + finalFee + reward + (disputerBond / 2);
        // Transfer to proposer
    } else {
        // Disputer was correct
        payout = disputerBond + finalFee + (proposerBond / 2);
        // Transfer to disputer
    }
    // Other half of loser's bond was already burned in disputePrice()
}
```

---

### 5. settleAndGetPrice()

**Purpose:** Settle and return the resolved price in one call

```solidity
function settleAndGetPrice(
    bytes32 identifier,
    uint256 timestamp,
    bytes memory ancillaryData
) external returns (int256 price);
```

This is commonly used by adapter contracts like UmaCtfAdapter:
```solidity
// In UmaCtfAdapter.resolve()
int256 price = optimisticOracle.settleAndGetPrice(
    identifier,
    timestamp,
    ancillaryData
);
// Convert price to CTF payout
```

---

### 6. Configuration Functions

```solidity
// Set custom bond amount
function setBond(
    bytes32 identifier,
    uint256 timestamp,
    bytes memory ancillaryData,
    uint256 bond
) external returns (uint256 totalBond);

// Set custom liveness period
function setCustomLiveness(
    bytes32 identifier,
    uint256 timestamp,
    bytes memory ancillaryData,
    uint256 customLiveness
) external;

// Enable callbacks
function setCallbacks(
    bytes32 identifier,
    uint256 timestamp,
    bytes memory ancillaryData,
    bool callbackOnPriceProposed,
    bool callbackOnPriceDisputed,
    bool callbackOnPriceSettled
) external;

// Set event-based mode
function setEventBased(
    bytes32 identifier,
    uint256 timestamp,
    bytes memory ancillaryData
) external;
```

---

## Bond Mechanics & Economics

### Bond Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           BOND MECHANICS                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PROPOSAL:                                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ Proposer deposits: bond + finalFee                                  │    │
│  │ Example: 750 USDC + 5 USDC = 755 USDC                              │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  DISPUTE (if occurs):                                                       │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ Disputer deposits: bond + finalFee                                  │    │
│  │ Example: 750 USDC + 5 USDC = 755 USDC                              │    │
│  │                                                                      │    │
│  │ Total locked: 1,510 USDC (both bonds)                               │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  SETTLEMENT OUTCOMES:                                                       │
│                                                                             │
│  Case 1: No Dispute (Optimistic Settlement)                                │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ Proposer receives: bond + finalFee + reward                         │    │
│  │ Example: 750 + 5 + 5 = 760 USDC                                     │    │
│  │ Net profit: +5 USDC (reward)                                        │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  Case 2: Dispute - Proposer Wins                                           │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ Proposer receives: bond + finalFee + reward + (disputerBond / 2)    │    │
│  │ Example: 750 + 5 + 5 + 375 = 1,135 USDC                             │    │
│  │ Net profit: +380 USDC                                               │    │
│  │                                                                      │    │
│  │ Disputer loses: full bond                                           │    │
│  │ Protocol receives: disputerBond / 2 = 375 USDC                      │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  Case 3: Dispute - Disputer Wins                                           │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ Disputer receives: bond + finalFee + (proposerBond / 2)             │    │
│  │ Example: 750 + 5 + 375 = 1,130 USDC                                 │    │
│  │ Net profit: +375 USDC                                               │    │
│  │                                                                      │    │
│  │ Proposer loses: full bond + reward forfeited                        │    │
│  │ Protocol receives: proposerBond / 2 = 375 USDC                      │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Default Parameters

| Parameter | Polymarket Default | Description |
|-----------|-------------------|-------------|
| **Bond** | 750 USDC | Stake required from proposers |
| **Liveness** | 7,200 seconds (2 hours) | Dispute window |
| **Final Fee** | ~5 USDC | Protocol fee |
| **Reward** | ~5 USDC | Proposer incentive |

### Game Theory

The system creates a **Schelling Point** where:
- Honest proposers are rewarded
- Incorrect proposers lose their bond
- Frivolous disputes are expensive (lose bond)
- Valid disputes are profitable (win half of wrong proposer's bond)

**Expected Value Analysis:**
```
For honest proposer:
  EV = P(no dispute) × reward + P(dispute) × P(win) × (reward + disputerBond/2)
     - P(dispute) × P(lose) × bond

With 98% non-dispute rate and honest majority:
  EV ≈ 0.98 × 5 + 0.02 × 0.95 × 380 - 0.02 × 0.05 × 750
     ≈ 4.9 + 7.22 - 0.75
     ≈ +11.37 USDC per proposal (positive expected value for truth-telling)
```

### Economic Security: Why No Timestamp Validation?

**Key Insight:** UMA relies on **economic incentives** instead of **code-based validation** for security.

**Example Edge Case: Premature Proposals**

Consider a market "Will X happen by Day 3?" where someone proposes "YES" on Day 0 (before the event could possibly happen).

**Why It's Allowed:**

**File:** [OptimisticOracleV2.sol - proposePriceFor() L474-500](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol#L474-L500)

```solidity
function proposePriceFor(
    address proposer,
    address requester,
    bytes32 identifier,
    uint256 timestamp,
    bytes memory ancillaryData,
    int256 proposedPrice
) public override nonReentrant() returns (uint256 totalBond) {
    require(proposer != address(0));

    // ✅ Only validation: Must be in Requested state
    require(_getState(...) == State.Requested);

    // ❌ NO checks for:
    // - Event timestamp in ancillaryData
    // - Whether enough time has passed
    // - Whether real-world conditions could be met yet

    request.proposer = proposer;
    request.proposedPrice = proposedPrice;
    request.expirationTime = getCurrentTime().add(liveness);

    // Accept bond
    _pullBond(proposer, request);
}
```

**Why This Design Works:**

| Actor | Economic Incentive | Result |
|-------|-------------------|--------|
| **Premature Proposer** | Loses 350 USDC when disputed | Won't propose early ❌ |
| **Rational Disputer** | Earns 405 USDC for catching error | Actively monitors ✅ |
| **Honest Proposer** | Earns 5 USDC for correct answer | Waits for actual result ✅ |

**Money Flow on Premature Proposal:**

```
Day 0: Proposer submits "YES" (750 USDC bond)
       ├─ Event hasn't occurred yet
       └─ Technically valid (no timestamp check)

Day 0: Disputer challenges (750 USDC bond)
       ├─ "Event hasn't happened, can't be YES"
       └─ Triggers dispute

Outcome: First Dispute
       ├─ Market resets to "Requested" state
       ├─ Proposer loses 350 USDC (50% burned + 50% to disputer)
       ├─ Disputer gains 405 USDC (their 750 + 400 from proposer + 5 reward)
       └─ New proposal can be submitted after event

Day 3: Event occurs (or doesn't)
       ├─ Valid proposal submitted
       └─ Market resolves correctly ✅
```

**Code Reference - First Dispute Resets Market:**

**File:** [UmaCtfAdapter.sol - priceDisputed() L289-310](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L289-L310)

```solidity
function priceDisputed(
    bytes32 identifier,
    uint256 timestamp,
    bytes memory ancillaryData,
    uint256 refund
) external {
    bytes32 questionID = keccak256(ancillaryData);
    QuestionData storage questionData = questions[questionID];

    // ✅ First dispute: RESET the market
    if (!questionData.reset) {
        _reset(
            msg.sender,
            questionID,
            refund > 0,
            questionData
        );
        return;
    }

    // Second dispute: Escalate to DVM (no reset)
    // ...
}

function _reset(...) internal {
    questionData.reset = true;
    questionData.requestTimestamp = block.timestamp;

    // Create NEW price request
    _requestPrice(
        requestor,
        block.timestamp,
        questionData.ancillaryData,  // Same question
        questionData.rewardToken,
        questionData.reward,
        questionData.proposalBond,
        questionData.liveness
    );

    // Market returns to State: Requested
    // Now correct answer can be proposed
}
```

### Design Philosophy: Economic Security > Code Validation

**Traditional Oracle Approach:**
```solidity
// ❌ What UMA does NOT do
require(block.timestamp >= eventTimestamp, "Too early");
require(verifyRealWorldConditions(), "Event not met");
```

**UMA's Approach:**
```solidity
// ✅ What UMA does instead
// 1. Anyone can propose anything
// 2. Economic cost: 750 USDC bond
// 3. Dispute mechanism: Others can challenge
// 4. Wrong proposals lose 350 USDC
// 5. Correct disputes earn 405 USDC
// → Result: Economic rationality ensures correctness
```

**Why This Works:**

1. **Smart contracts can't verify real-world events**
   - No way to check if "Bitcoin reached $100k"
   - No way to verify if "election happened on date X"
   - Needs human input (proposers)

2. **Economic disincentives are stronger than code checks**
   - Code can be gamed or have edge cases
   - Losing 350 USDC is universally understood
   - Earning 405 USDC incentivizes active monitoring

3. **Dispute mechanism provides security**
   - 2-hour liveness gives time to dispute
   - Anyone can dispute (permissionless)
   - First dispute resets market (no harm done)
   - Second dispute goes to DVM (full arbitration)

4. **MOOV2 adds reputation layer** (since August 2025)
   - Only whitelisted proposers can submit
   - Must maintain >95% accuracy
   - Lose access if accuracy drops
   - Reduces premature/incorrect proposals

### Attack Prevention: Economic Rationality

**Attack Scenario: Premature Proposal Attack**

```
Attacker's Goal: Resolve market before event occurs
Attacker's Action: Propose "YES" immediately

Expected Outcome for Attacker:
├─ Post 750 USDC bond
├─ Wait 2 hours for liveness
├─ Call resolve() and profit
└─ Total cost: Just gas fees

Actual Outcome:
├─ Rational disputer sees obvious error
├─ Disputer posts 750 USDC bond
├─ Dispute triggered
├─ Market resets
├─ Attacker loses 350 USDC
└─ Attack fails ❌

Why Attack Failed:
├─ Disputer profit: +405 USDC (strong incentive to monitor)
├─ Attacker loss: -350 USDC (strong disincentive)
└─ Economic rationality prevents attack
```

**Defense Summary:**

| Defense Layer | Mechanism | Effectiveness |
|---------------|-----------|--------------|
| **Economic Penalty** | -350 USDC for wrong proposals | Strong disincentive |
| **Economic Reward** | +405 USDC for catching errors | Strong monitoring incentive |
| **Time Delay** | 2-hour liveness period | Gives time for disputes |
| **State Machine** | First dispute resets market | Prevents incorrect resolution |
| **Reputation** | MOOV2 whitelist (>95% accuracy) | Trusted proposers only |
| **Escalation** | Second dispute → DVM voting | Ultimate arbitration |

**The Result:** No need for on-chain timestamp validation or real-world event verification. Economic incentives ensure proposers only submit answers they're confident are correct.

---

## DVM Escalation & Voting

### When DVM Gets Involved

The DVM (Data Verification Mechanism) is triggered when:
1. A proposal is disputed via `disputePrice()`
2. OOv2 calls `oracle.requestPrice()` to escalate

### DVM Voting Options

For YES_OR_NO_QUERY requests, DVM voters can select:

| Option | Value | Meaning |
|--------|-------|---------|
| **p1** | 0 | NO |
| **p2** | 1e18 | YES |
| **p3** | 0.5e18 | UNKNOWN / 50-50 split |
| **p4** | Magic value | TOO EARLY / Cannot be determined yet |

### The P4 Mechanism: Disputing Premature Proposals

**P4 ("Too Early")** is a special voting option for when a proposal is made before the outcome can be determined.

**Key Insight:** Disputers don't need to know the final answer. They can challenge a premature proposal purely on timing grounds.

```
Example:
├─ Market: "Will X happen by Day 3?"
├─ Day 0: Proposer submits "YES" (event hasn't occurred!)
├─ Day 0: Disputer challenges (doesn't claim "NO is correct")
├─ DVM Vote: Voters select P4 ("Too Early")
├─ Result: Proposer LOSES bond regardless of eventual outcome
└─ Market: Resets for proper proposal later
```

**Why P4 Exists:**
> "The purpose of proposing data is to report on an event that has already happened. Submitting proposals too early isn't reporting—it's predicting."

**Smart Contract Enforcement:**
```solidity
// Proposers CANNOT propose P4 themselves
require(proposedPrice != TOO_EARLY_RESPONSE, "Cannot propose TOO_EARLY");
```

This prevents proposers from gaming the system by proposing "too early" on themselves.

### Winner Determination: Binary Check

**Critical:** The DVM outcome is a binary check—does it match the proposer's answer?

```solidity
bool disputeSuccess = request.resolvedPrice != request.proposedPrice;

if (disputeSuccess) {
    // Disputer wins (DVM disagreed with proposer)
    recipient = request.disputer;
} else {
    // Proposer wins (DVM agreed with proposer)
    recipient = request.proposer;
}
```

**Implications:**

| Proposer Says | DVM Votes | Winner | Why |
|---------------|-----------|--------|-----|
| YES | YES | Proposer | Match |
| YES | NO | Disputer | No match |
| YES | UNKNOWN | Disputer | No match |
| YES | P4 | Disputer | No match |
| NO | UNKNOWN | Disputer | No match |

**Key Insight:** Disputers are rewarded for identifying wrong proposals, not for predicting right ones.

See [resolution-qa.md](resolution-qa.md#dvm-votes-a-third-answer-who-loses) for detailed scenarios.

### VotingV2 Contract

**GitHub:** [VotingV2.sol](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/data-verification-mechanism/implementation/VotingV2.sol)

**Key Data Structures:**

```solidity
struct PriceRequest {
    uint32 lastVotingRound;         // Last round voted on
    bool isGovernance;              // Governance vs price request
    uint64 time;                    // Timestamp for evaluation
    bytes32 identifier;             // Price identifier
    mapping voteInstances;          // Votes across rounds
    bytes ancillaryData;            // Context data
}

struct VoteSubmission {
    bytes32 commit;                 // Hash of vote (zeroed after reveal)
    bytes32 revealHash;             // Hash for reward computation
}
```

### Commit-Reveal Voting Process

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DVM VOTING PROCESS                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PHASE 1: COMMIT (24-48 hours)                                             │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ Voters submit hash of their vote:                                   │    │
│  │                                                                      │    │
│  │ bytes32 commitHash = keccak256(abi.encodePacked(                    │    │
│  │     price,           // The voted price                             │    │
│  │     salt,            // Random salt for privacy                     │    │
│  │     voter,           // Voter address                               │    │
│  │     time,            // Request timestamp                           │    │
│  │     ancillaryData,   // Question data                               │    │
│  │     roundId,         // Current voting round                        │    │
│  │     identifier       // Price identifier                            │    │
│  │ ));                                                                  │    │
│  │                                                                      │    │
│  │ voting.commitVote(identifier, time, ancillaryData, commitHash);     │    │
│  │                                                                      │    │
│  │ ✓ Votes remain private (only hash visible)                          │    │
│  │ ✓ Prevents vote copying/front-running                               │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│                                    ▼                                        │
│  PHASE 2: REVEAL (24-48 hours)                                             │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ Voters reveal their actual vote:                                    │    │
│  │                                                                      │    │
│  │ voting.revealVote(                                                  │    │
│  │     identifier,                                                     │    │
│  │     time,                                                           │    │
│  │     price,           // Actual voted price                          │    │
│  │     ancillaryData,                                                  │    │
│  │     salt             // Same salt used in commit                    │    │
│  │ );                                                                   │    │
│  │                                                                      │    │
│  │ Contract verifies: hash(revealed) == committed hash                 │    │
│  │                                                                      │    │
│  │ ✓ Invalid reveals are rejected                                      │    │
│  │ ✓ Votes tallied by stake weight                                     │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│                                    ▼                                        │
│  PHASE 3: RESOLUTION                                                        │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ Quorum Requirements:                                                │    │
│  │ • GAT: Minimum 5 million UMA tokens must vote                       │    │
│  │ • SPAT: ≥65% of staked UMA must participate and agree               │    │
│  │                                                                      │    │
│  │ Outcome:                                                            │    │
│  │ • Majority price becomes resolvedPrice                              │    │
│  │ • Correct voters: Keep stake + earn from slashed                    │    │
│  │ • Wrong voters: Slashed 0.1% per incorrect vote                     │    │
│  │ • Non-voters: Slashed (inactivity penalty)                          │    │
│  │                                                                      │    │
│  │ If quorum not met: Vote rolls to next round (max 4 rolls)           │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Slashing Economics

```solidity
// DVM 2.0 slashing parameters
uint256 constant WRONG_VOTE_SLASH = 0.001e18;  // 0.1% per wrong vote
uint256 constant MISSED_VOTE_SLASH = 0.001e18; // 0.1% per missed vote
```

**Example:**
```
Voter stakes: 10,000 UMA
Votes incorrectly on 1 dispute out of 10 rounds

Slashed: 10,000 × 0.1% = 10 UMA
Remaining stake: 9,990 UMA

Slashed UMA redistributed to correct voters
```

### Price Resolution Flow

```solidity
// Inside OOv2 settle() when state == Resolved
function _getResolvedPrice() internal returns (int256) {
    OracleInterface oracle = _getOracle();
    return oracle.getPrice(identifier, timestamp, ancillaryData);
}
```

---

## Production Transaction Examples

### Verified Contract (Polygon)

**OptimisticOracleV2:** [`0xee3afe347d5c74317041e2618c49534daf887c24`](https://polygonscan.com/address/0xee3afe347d5c74317041e2618c49534daf887c24#code)

- **Status:** ✅ Verified
- **Compiler:** Solidity v0.8.9
- **Optimization:** Enabled (1,000,000 runs)
- **License:** GNU GPLv3

### Example: ProposePrice Transaction

**Transaction:** [`0x5acde6219982035a07b5ac5ed57e15573b595a302f17bda4fa4c631147f8fa7f`](https://polygonscan.com/tx/0x5acde6219982035a07b5ac5ed57e15573b595a302f17bda4fa4c631147f8fa7f)

```
Block:          81,796,595
Timestamp:      Jan 18, 2026, 05:12:42 AM UTC
From:           0x7F56658341f1C660D9C93B01a6B087e39e84d456
To:             0xeE3Afe347D5C74317041E2618C49534dAf887c24 (OOv2)
Value:          0 POL
Gas Used:       169,910
Transaction Fee: 0.01694 POL (~$0.002)

Function: proposePrice(
    address requester,
    bytes32 identifier,
    uint256 timestamp,
    bytes ancillaryData,
    int256 proposedPrice
)

Token Transfer: 750 USDC.e → OOv2 Contract (bond)

ProposePrice Event:
├── Requester:   0x2F5e3684cb1F318ec51b00Edba38d79Ac2c0aA9d (UmaCtfAdapter V3)
├── Proposer:    0x7F56658341f1C660D9C93B01a6B087e39e84d456
├── Identifier:  YES_OR_NO_QUERY
├── Timestamp:   1767045037
├── Proposed:    0 (NO)
├── Expiration:  1768720362
└── Currency:    USDC.e

Ancillary Data (decoded):
"q: Will Grok 4.20 be released by January 31, 2026?
 p1: No, p2: Yes, p3: Unknown"
```

### Example: Settle Transaction

**Transaction:** [`0xacae1954bd34e64f2de81fa1eed1fff1d538b88a8a522c37d215251f627d7e5c`](https://polygonscan.com/tx/0xacae1954bd34e64f2de81fa1eed1fff1d538b88a8a522c37d215251f627d7e5c)

```
Block:          81,796,711
Timestamp:      Jan 18, 2026, 05:16:34 AM UTC
From:           0x33965F7D08F61A62B86C1Ab9Be5d82C42F4c3081
To:             0xeE3Afe347D5C74317041E2618C49534dAf887c24 (OOv2)
Value:          0 POL
Gas Used:       134,879
Transaction Fee: 0.0138 POL (~$0.002)

Function: settle(
    address requester,
    bytes32 identifier,
    uint256 timestamp,
    bytes ancillaryData
)

Token Transfer: 755 USDC.e → 0xc9Ffd41C...2BFb6b04E (proposer payout)

Settle Event:
├── Requester:   0x2F5e3684cb1F318ec51b00Edba38d79Ac2c0aA9d
├── Proposer:    0xc9Ffd41C7500d34606606246D8fb6242BFb6b04E
├── Disputer:    0x0000000000000000000000000000000000000000 (none)
├── Identifier:  YES_OR_NO_QUERY
├── Timestamp:   1768474888
├── Final Price: 0 (NO)
└── Payout:      755,000,000 (755 USDC.e = bond + fee + reward)

Ancillary Data:
"Highest temperature in Buenos Aires on January 17, 2026"
```

### Dispute Statistics (Historical)

As of July 2024:
- **Total markets settled:** 11,093
- **Disputes raised:** 217
- **Dispute rate:** ~2%
- **98% settled optimistically** (no dispute)

---

## Contract Addresses

### Polygon Mainnet

| Contract | Address | Verified |
|----------|---------|----------|
| **OptimisticOracleV2** | [`0xee3afe347d5c74317041e2618c49534daf887c24`](https://polygonscan.com/address/0xee3afe347d5c74317041e2618c49534daf887c24) | ✅ |
| **OptimisticOracleV3** | [`0x5953f2538F613E05bAED8A5AeFa8e6622467AD3D`](https://polygonscan.com/address/0x5953f2538F613E05bAED8A5AeFa8e6622467AD3D) | ✅ |
| **Store** | [`0xE58480CA74f1A819faFd777BEDED4E2D5629943d`](https://polygonscan.com/address/0xE58480CA74f1A819faFd777BEDED4E2D5629943d) | ✅ |
| **Finder** | [`0x09aea4b2242abC8bb4BB78D537A67a245A7bEC64`](https://polygonscan.com/address/0x09aea4b2242abC8bb4BB78D537A67a245A7bEC64) | ✅ |
| **VotingToken (UMA)** | [`0x3066818837c5e6eD6601bd5a91B0762877A6B731`](https://polygonscan.com/address/0x3066818837c5e6eD6601bd5a91B0762877A6B731) | ✅ |

### Polymarket Adapter Contracts (Polygon)

| Version | Address | Description |
|---------|---------|-------------|
| **UmaCtfAdapter V3** | [`0x2F5e3684cb1F318ec51b00Edba38d79Ac2c0aA9d`](https://polygonscan.com/address/0x2F5e3684cb1F318ec51b00Edba38d79Ac2c0aA9d) | Current production adapter |
| **UmaCtfAdapter V2** | [`0x6A9D222616C90FcA5754cd1333cFD9b7fb6a4F74`](https://polygonscan.com/address/0x6A9D222616C90FcA5754cd1333cFD9b7fb6a4F74) | Previous version |
| **UmaCtfAdapter V1** | [`0xCB1822859cEF82Cd2Eb4E6276C7916e692995130`](https://polygonscan.com/address/0xCB1822859cEF82Cd2Eb4E6276C7916e692995130) | Original adapter |

### Ethereum Mainnet

| Contract | Address |
|----------|---------|
| **OptimisticOracleV2** | `0xA0Ae6609447e57a42c51B50EAe921D701823FFAe` |
| **VotingV2 (DVM)** | `0x004395edb43EFca9885CEdad51EC9fAf93Bd34ac` |

---

## Price Identifiers

### YES_OR_NO_QUERY (UMIP-107)

Used for binary questions:

| Price | Meaning | Payout Vector | YES Token | NO Token |
|-------|---------|---------------|-----------|----------|
| `0` | NO | [0, 1] | $0.00 | $1.00 |
| `1e18` | YES | [1, 0] | $1.00 | $0.00 |
| `0.5e18` | UNKNOWN | [1, 1] | $0.50 | $0.50 |

### UNKNOWN Resolution (0.5e18)

UNKNOWN is a valid resolution outcome when:

1. **Ambiguous question wording** - The question doesn't clearly map to what happened
2. **Subjective criteria** - No objective answer exists
3. **Event didn't occur as specified** - Conditions for YES or NO weren't met
4. **Contradictory evidence** - Multiple valid interpretations exist
5. **Technical nullification** - Event was cancelled/voided

**Important Distinctions:**
- **UNKNOWN (p3):** For genuinely ambiguous outcomes where neither YES nor NO is correct
- **TOO EARLY (p4):** For proposals made before the outcome can be determined

**NegRisk Limitation:** UNKNOWN resolution (`[1,1]` payout) is NOT valid for NegRisk markets (multi-outcome markets). These markets require exactly one winner.

See [resolution-qa.md](resolution-qa.md#unknown-resolution-when-real-world-events-are-ambiguous) for detailed scenarios.

### Ancillary Data Format

```
q: [Question text]
p1: [First outcome label]
p2: [Second outcome label]
p3: [Unknown/Invalid label]
res_data: [Resolution source URL]
```

**Example:**
```
q: Will ETH price be above $5000 on December 31, 2026, 00:00 UTC?
p1: Yes
p2: No
p3: Unknown
res_data: https://www.coingecko.com/en/coins/ethereum
```

---

## Managed Optimistic Oracle V2 (MOOV2)

As of August 2025, Polymarket transitioned to MOOV2 (UMIP-189):

### Key Changes from Standard OOv2

| Feature | Standard OOv2 | MOOV2 |
|---------|---------------|-------|
| **Who can propose** | Anyone | Whitelisted addresses only |
| **Who can dispute** | Anyone | Anyone (unchanged) |
| **Initial whitelist** | N/A | 37 addresses |

### Whitelist Criteria
- Risk Labs / Polymarket employees
- Users with 20+ proposals and >95% accuracy
- Historical track record required

### Benefits
- Higher due diligence on proposals
- Fewer incorrect proposals on non-contentious markets
- Maintained decentralization via open disputes

---

## Events

### OptimisticOracleV2 Events

```solidity
event RequestPrice(
    address indexed requester,
    bytes32 identifier,
    uint256 timestamp,
    bytes ancillaryData,
    address currency,
    uint256 reward,
    uint256 finalFee
);

event ProposePrice(
    address indexed requester,
    address indexed proposer,
    bytes32 identifier,
    uint256 timestamp,
    bytes ancillaryData,
    int256 proposedPrice,
    uint256 expirationTimestamp,
    address currency
);

event DisputePrice(
    address indexed requester,
    address indexed proposer,
    address indexed disputer,
    bytes32 identifier,
    uint256 timestamp,
    bytes ancillaryData,
    int256 proposedPrice
);

event Settle(
    address indexed requester,
    address indexed proposer,
    address indexed disputer,
    bytes32 identifier,
    uint256 timestamp,
    bytes ancillaryData,
    int256 price,
    uint256 payout
);
```

---

## References

### Internal Documentation
- [Resolution Q&A](resolution-qa.md) - Comprehensive Q&A on resolution flow, disputes, and edge cases
- [Market Lifecycle](market-lifecycle.md) - High-level overview of market phases
- [Market Creation Deep Dive](market-creation-deep-dive.md) - Contract-level walkthrough with transaction examples

### Official Documentation
- [UMA: How Oracle Works](https://docs.uma.xyz/protocol-overview/how-does-umas-oracle-work)
- [UMA: Optimistic Oracle V2](https://docs.uma.xyz/developers/optimistic-oracle)
- [UMA: Optimistic Oracle V3](https://docs.uma.xyz/developers/optimistic-oracle-v3)
- [UMA: DVM 2.0](https://docs.uma.xyz/protocol-overview/dvm-2.0)
- [UMA: Network Addresses](https://docs.uma.xyz/resources/network-addresses)
- [Polymarket: UMA Resolution](https://docs.polymarket.com/developers/resolution/UMA)

### GitHub Repositories
- [UMAprotocol/protocol](https://github.com/UMAprotocol/protocol) - Core contracts
- [Polymarket/uma-ctf-adapter](https://github.com/Polymarket/uma-ctf-adapter) - Polymarket adapter

### UMIPs (UMA Improvement Proposals)
- [UMIP-107: YES_OR_NO_QUERY](https://github.com/UMAprotocol/UMIPs/blob/master/UMIPs/umip-107.md)
- [UMIP-183: MULTIPLE_VALUES](https://github.com/UMAprotocol/UMIPs/blob/master/UMIPs/umip-183.md)
- [UMIP-189: Managed Optimistic Oracle](https://github.com/UMAprotocol/UMIPs/blob/master/UMIPs/umip-189.md)

### Technical Articles
- [Inside UMA Oracle: How Prediction Markets Resolution Works (RockNBlock)](https://rocknblock.io/blog/how-prediction-markets-resolution-works-uma-optimistic-oracle-polymarket)
- [UMA Protocol: How does the popular Optimistic Oracle work? (MetaLamp)](https://metalamp.io/magazine/article/uma-protocol-how-does-the-popular-optimistic-oracle-work)

### Tools
- [UMA Oracle Dapp](https://oracle.uma.xyz/) - View and interact with proposals
- [Polygonscan: OOv2 Contract](https://polygonscan.com/address/0xee3afe347d5c74317041e2618c49534daf887c24)
