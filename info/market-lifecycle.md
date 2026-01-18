# Polymarket Market Lifecycle

> Complete flow from market creation to settlement

**Last Updated:** 2026-01-19

## Lifecycle Phases

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   CREATION   │──▶│   TRADING    │──▶│  RESOLUTION  │──▶│  SETTLEMENT  │
│              │   │              │   │              │   │              │
│ - Initialize │   │ - Split      │   │ - Propose    │   │ - Redeem     │
│ - Prepare    │   │ - Trade      │   │ - Dispute?   │   │ - Payout     │
│   Condition  │   │ - Merge      │   │ - Resolve    │   │              │
└──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
```

---

## Phase 1: Market Creation

### Prerequisites
- Market creator must be whitelisted by Polymarket
- USDC.e tokens for reward/bond

### Step 0: Approve USDC Spending

```solidity
// Contract: USDC.e
// Address: 0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174

function approve(
    address spender,    // UmaCtfAdapter address
    uint256 amount      // Reward amount in USDC (6 decimals)
) external returns (bool);
```

### Step 1: Initialize Market

**Source:** [UmaCtfAdapter.sol#L125-L167](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L125-L167)

```solidity
// Contract: UmaCtfAdapter
// Address: 0x2F5e3684cb1F318ec51b00Edba38d79Ac2c0aA9d (v3)

function initialize(
    bytes memory ancillaryData,  // Encoded question + resolution rules
    address rewardToken,          // USDC.e address
    uint256 reward,               // Reward for correct proposer (e.g., 5 USDC)
    uint256 proposalBond,         // Bond amount (0 = default ~750 USDC)
    uint256 liveness              // Dispute window (0 = default 7200 seconds)
) external returns (
    bytes32 questionId,
    bytes32 conditionId
);
```

**What happens internally:**
1. Stores question parameters on adapter
2. Calls `prepareCondition()` on CTF contract
3. Calls `requestPrice()` on UMA Optimistic Oracle

### ancillaryData Format

The `ancillaryData` is a UTF-8 encoded string containing:
- Question text
- Resolution rules
- Market metadata (title, description, market_id)

```
Example (hex-encoded):
q: Will Bitcoin reach $100,000 by December 31, 2025?
res_data: p1: 0, p2: 1, p3: 0.5 (for tie/invalid)
...
```

### Derived IDs

```solidity
// questionId = keccak256(ancillaryData)
questionId = keccak256(abi.encodePacked(ancillaryData));

// conditionId = keccak256(oracle, questionId, outcomeSlotCount)
conditionId = keccak256(abi.encodePacked(
    oracle,           // UmaCtfAdapter address
    questionId,
    uint256(2)        // Always 2 for binary markets
));
```

---

## Phase 2: Trading

### Step 2: Approve CTF for Splitting

```solidity
// Contract: USDC.e
function approve(
    address spender,    // CTF: 0x4D97DCd97eC945f40cF65F87097ACe5EA0476045
    uint256 amount      // Max amount or type(uint256).max
) external returns (bool);
```

### Step 3: Split Position (Mint Outcome Tokens)

**Source:** [ConditionalTokens.sol#L105-L159](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol#L105-L159)

```solidity
// Contract: ConditionalTokens (CTF)
// Address: 0x4D97DCd97eC945f40cF65F87097ACe5EA0476045

function splitPosition(
    IERC20 collateralToken,        // USDC.e address
    bytes32 parentCollectionId,    // 0x0 for root positions
    bytes32 conditionId,           // From initialization
    uint256[] partition,           // [1, 2] for binary (YES, NO)
    uint256 amount                 // USDC amount to split
) external;
```

**Effect:**
- Burns `amount` USDC from caller
- Mints `amount` YES tokens (indexSet=1)
- Mints `amount` NO tokens (indexSet=2)

### Step 4: Approve Exchange for Trading

```solidity
// Contract: ConditionalTokens (CTF)
function setApprovalForAll(
    address operator,   // CTF Exchange: 0x4bFb41d5B3570DeFd03C39A9A4D8dE6Bd8B8982E
    bool approved       // true
) external;
```

### Trading via CLOB

Orders are EIP-712 signed off-chain and matched by the operator:

```typescript
interface Order {
    salt: bigint;           // Random nonce
    maker: address;         // Order creator
    signer: address;        // Signing key
    taker: address;         // 0x0 for any taker
    tokenId: bigint;        // Position ID (ERC1155 token)
    makerAmount: bigint;    // Amount offered
    takerAmount: bigint;    // Amount requested
    expiration: bigint;     // Unix timestamp
    nonce: bigint;          // For cancellation
    feeRateBps: number;     // Fee rate signed into order
    side: 'BUY' | 'SELL';
    signatureType: number;
}
```

### Matching Modes

| Mode | Description |
|------|-------------|
| **NORMAL** | Direct token-for-token swap |
| **MINT** | Complementary orders create new tokens from collateral |
| **MERGE** | Complementary sell orders merge tokens back to collateral |

### Merge Positions (Optional)

```solidity
// Contract: ConditionalTokens (CTF)
function mergePositions(
    IERC20 collateralToken,
    bytes32 parentCollectionId,    // 0x0
    bytes32 conditionId,
    uint256[] partition,           // [1, 2]
    uint256 amount
) external;
```

**Effect:** Burns equal amounts of YES and NO tokens, returns USDC

---

## Phase 3: Resolution

### Resolution Paths

```
Path 1 (No Dispute):
Initialize ──▶ Propose ──▶ [2hr liveness] ──▶ Resolve

Path 2 (Single Dispute):
Initialize ──▶ Propose ──▶ Dispute ──▶ Propose ──▶ [2hr] ──▶ Resolve

Path 3 (DVM Escalation):
Initialize ──▶ Propose ──▶ Dispute ──▶ Propose ──▶ Dispute ──▶ DVM Vote ──▶ Resolve
```

### Step 5: Propose Price (Anyone)

**Source:** [OptimisticOracleV2.sol#L462-L508](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol#L462-L508)

```solidity
// Contract: OptimisticOracleV2
// Address: 0xee3afe347d5c74317041e2618c49534daf887c24

function proposePrice(
    address requester,      // UmaCtfAdapter address
    bytes32 identifier,     // "YES_OR_NO_QUERY" identifier
    uint256 timestamp,      // Request timestamp
    bytes ancillaryData,    // Question data
    int256 proposedPrice    // 1e18 = YES, 0 = NO
) external returns (uint256 totalBond);
```

**Bond Requirement:** ~750 USDC (default)
**Reward:** 5 USDC if proposal is accepted

#### Timing: When Can You Propose?

**Answer: IMMEDIATELY - No waiting period required.**

**Code Evidence:** [OptimisticOracleV2.sol - proposePriceFor() L474-500](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol#L474-L500)

```solidity
function proposePriceFor(...) public override returns (uint256 totalBond) {
    require(proposer != address(0));

    // ✅ Only check: Must be in Requested state
    require(_getState(...) == State.Requested);

    // ❌ NO time validation
    // ❌ NO event timestamp check
    // ❌ NO "wait until event occurs" requirement

    request.proposer = proposer;
    request.proposedPrice = proposedPrice;
    request.expirationTime = getCurrentTime().add(liveness);

    _pullBond(proposer, request);
}
```

**What prevents premature proposals?**

Not code validation, but **economic incentives**:

| Scenario | Proposer Action | Economic Outcome |
|----------|----------------|------------------|
| **Valid Proposal** | Propose correct answer after event | +5 USDC reward ✅ |
| **Premature Proposal** | Propose before event occurs | -350 USDC loss (disputed) ❌ |
| **Rational Disputer** | Dispute premature proposal | +405 USDC profit ✅ |

**Example Timeline:**

```
Day 0: Market created "Will X happen by Day 3?"
       └─> Anyone CAN propose immediately (no code restriction)
       └─> But loses 350 USDC if disputed for being premature

Day 3: Event occurs (or doesn't)
       └─> Safe to propose now (answer is verifiable)
       └─> Earn 5 USDC if correct and undisputed
```

**Protection Mechanisms:**
1. **Economic Disincentive:** -350 USDC for wrong proposals
2. **Economic Incentive:** +405 USDC for disputing wrong proposals
3. **2-Hour Liveness:** Gives time for disputes to occur
4. **First Dispute Resets Market:** No harm from premature proposals
5. **MOOV2 Whitelist** (since Aug 2025): Only trusted proposers (>95% accuracy)

See [resolution-qa.md](resolution-qa.md#edge-case-premature-proposals) for detailed analysis.

### Proposed Price Values

| Value | Meaning |
|-------|---------|
| `1e18` (1 * 10^18) | YES outcome wins |
| `0` | NO outcome wins |
| `0.5e18` | Invalid/tie (both tokens worth 50%) |

### Step 6: Dispute Price (Optional)

**Source:** [OptimisticOracleV2.sol#L527-L594](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol#L527-L594)

```solidity
// Contract: OptimisticOracleV2
function disputePrice(
    address requester,
    bytes32 identifier,
    uint256 timestamp,
    bytes ancillaryData
) external returns (uint256 totalBond);
```

**First Dispute:** Resets market, new proposal requested
**Second Dispute:** Escalates to UMA DVM (48-96 hour voting period)

#### Disputing Premature Proposals: The P4 Mechanism

When disputing a proposal made before the event could occur, disputers don't need to know the final answer. UMA provides a special **P4** voting option ("Too Early") for this scenario.

**DVM Voting Options:**
- **p1**: YES (1e18)
- **p2**: NO (0)
- **p3**: UNKNOWN/50-50 (0.5e18)
- **p4**: TOO EARLY / Cannot be determined yet

**Key Rule:** Proposers cannot submit P4 themselves. If a proposal is disputed as premature, the proposer loses their bond even if their prediction turns out to be correct.

See [resolution-qa.md](resolution-qa.md#disputing-premature-proposals-the-p4-mechanism) for detailed analysis.

### Step 7: Resolve Market

**Source:** [UmaCtfAdapter.sol#L169-L179](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L169-L179)

```solidity
// Contract: UmaCtfAdapter
function resolve(bytes32 questionId) external;
```

**What happens internally:**
1. Fetches settled price from Optimistic Oracle
2. Converts price to payout vector `[1,0]` or `[0,1]`
3. Calls `reportPayouts()` on CTF contract

---

## Phase 4: Settlement

### Step 8: Redeem Positions

**Source:** [ConditionalTokens.sol#L210-L254](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol#L210-L254)

```solidity
// Contract: ConditionalTokens (CTF)
function redeemPositions(
    IERC20 collateralToken,        // USDC.e
    bytes32 parentCollectionId,    // 0x0
    bytes32 conditionId,
    uint256[] indexSets            // [1] for YES, [2] for NO, or [1,2] for both
) external;
```

**Effect:**
- Burns outcome tokens held by caller
- Transfers proportional USDC based on payout vector

### Payout Vectors

| Outcome | Payout Vector | YES Token Value | NO Token Value |
|---------|---------------|-----------------|----------------|
| YES wins | `[1, 0]` | 1 USDC | 0 USDC |
| NO wins | `[0, 1]` | 0 USDC | 1 USDC |
| Invalid/UNKNOWN | `[1, 1]` | 0.5 USDC | 0.5 USDC |

**Note:** `[1, 1]` payout is incompatible with NegRisk markets.

### UNKNOWN Resolution

UNKNOWN (0.5e18) is a valid resolution outcome when:
- Question wording is genuinely ambiguous
- Event was cancelled or didn't occur as specified
- Contradictory evidence exists
- Subjective criteria cannot be objectively evaluated

**Important:** UNKNOWN is for genuinely ambiguous outcomes, NOT for "too early to know" scenarios. Use P4 disputes for premature proposals instead.

See [resolution-qa.md](resolution-qa.md#unknown-resolution-when-real-world-events-are-ambiguous) for detailed scenarios.

---

## Position ID Calculation

```solidity
// 1. Calculate conditionId
bytes32 conditionId = keccak256(abi.encodePacked(
    oracle,
    questionId,
    uint256(2)  // outcomeSlotCount
));

// 2. Calculate collectionId for each outcome
bytes32 collectionIdYes = CTF.getCollectionId(
    bytes32(0),     // parentCollectionId
    conditionId,
    1               // indexSet for YES
);

bytes32 collectionIdNo = CTF.getCollectionId(
    bytes32(0),
    conditionId,
    2               // indexSet for NO
);

// 3. Calculate positionId (ERC1155 tokenId)
uint256 positionIdYes = CTF.getPositionId(collateralToken, collectionIdYes);
uint256 positionIdNo = CTF.getPositionId(collateralToken, collectionIdNo);
```

---

## Timing Parameters

| Parameter | Default Value | Description |
|-----------|---------------|-------------|
| Liveness Period | 7,200 seconds (2 hours) | Time for disputes after proposal |
| DVM Voting | 48-96 hours | Time for UMA token holder voting |
| Bond Amount | ~750 USDC | Required stake for proposers/disputers |
| Proposer Reward | ~5-50 USDC | Reward for correct proposal (varies by market) |
| Final Fee | ~5 USDC | UMA protocol fee added to bond |

## Bond Economics

### Total Cost to Propose/Dispute
- **Bond:** 750 USDC (refundable if correct)
- **Final Fee:** ~5 USDC
- **Total:** ~755 USDC

### Settlement Payouts

| Scenario | Winner Receives | Loser Loses |
|----------|----------------|-------------|
| **No Dispute** | bond + finalFee + reward = ~760 USDC | N/A |
| **Dispute - Proposer Wins** | bond + finalFee + reward + (disputerBond/2) = ~1,180 USDC | Full bond (375 burned, 375 to winner) |
| **Dispute - Disputer Wins** | bond + finalFee + (proposerBond/2) = ~1,130 USDC | Full bond (375 burned, 375 to winner) |

**Note:** 50% of loser's bond is burned (sent to UMA Store) to prevent Sybil attacks.

See [resolution-qa.md](resolution-qa.md#finding-bond-costs-and-fees) for on-chain methods to query these values.

---

## References

### Internal Documentation
- [Resolution Q&A](resolution-qa.md) - Comprehensive Q&A on resolution flow, disputes, and edge cases
- [Market Creation Deep Dive](market-creation-deep-dive.md) - Contract-level walkthrough with transaction examples
- [UMA Oracle Deep Dive](uma-oracle.md) - Detailed analysis of UMA's Optimistic Oracle

### External Documentation
- [Polymarket Resolution Docs](https://docs.polymarket.com/developers/resolution/UMA)
- [UMA Optimistic Oracle Docs](https://docs.uma.xyz/developers/optimistic-oracle)
- [Gnosis CTF Developer Guide](https://conditional-tokens.readthedocs.io/en/latest/developer-guide.html)
- [UMA P4 Voting Explanation](https://blog.uma.xyz/articles/what-is-p4)
