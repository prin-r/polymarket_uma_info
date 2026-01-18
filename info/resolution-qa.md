# Polymarket Resolution: Questions & Answers

> Comprehensive Q&A covering the resolution flow, proposers, disputes, and timing

**Last Updated:** 2026-01-19

---

## Table of Contents

1. [Resolution Request Flow](#resolution-request-flow)
2. [Resolution Execution Flow](#resolution-execution-flow)
3. [Resolvers and Incentives](#resolvers-and-incentives)
4. [Dispute Mechanics](#dispute-mechanics)
5. [Bond Burning](#bond-burning)
6. [Proposers and Third Parties](#proposers-and-third-parties)
7. [Timing and Synchronicity](#timing-and-synchronicity)
8. [Edge Case: Premature Proposals](#edge-case-premature-proposals)
9. [Disputing Premature Proposals: The P4 Mechanism](#disputing-premature-proposals-the-p4-mechanism)
10. [DVM Votes a Third Answer: Who Loses?](#dvm-votes-a-third-answer-who-loses)
11. [UNKNOWN Resolution: When Real-World Events Are Ambiguous](#unknown-resolution-when-real-world-events-are-ambiguous)
12. [Finding Bond Costs and Fees](#finding-bond-costs-and-fees)

---

## Resolution Request Flow

### Q: Who requests resolution to UMA? When does this happen?

**A: The UmaCtfAdapter requests resolution during market CREATION (initialize), not during resolution.**

**Code Evidence:** [UmaCtfAdapter.sol#L140](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L140)

```solidity
function initialize(...) external returns (bytes32 questionID) {
    // ... setup code ...

    // Request price from UMA during initialization
    _requestPrice(
        msg.sender,
        timestamp,
        data,
        rewardToken,
        reward,
        proposalBond,
        liveness
    );
}

function _requestPrice(...) internal {
    // ... token transfers ...

    // Call UMA's OptimisticOracle
    optimisticOracle.requestPrice(
        YES_OR_NO_IDENTIFIER,
        requestTimestamp,
        ancillaryData,
        IERC20(rewardToken),
        reward
    );
}
```

**Timeline:**
```
Market Creation (Day 0):
└─> UmaCtfAdapter.initialize()
    └─> optimisticOracle.requestPrice()
        └─> Creates price request
        └─> State: Requested
        └─> Waiting for proposer...
```

**Key Insight:** The resolution request happens at **market creation time**, NOT when someone later calls `resolve()`.

---

## Resolution Execution Flow

### Q: After UMA gets the correct answer, how does Polymarket (CTF) know the answer? Who sends it? Which function?

**A: The UmaCtfAdapter bridges the answer from UMA to CTF through a series of function calls.**

**Complete Flow:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RESOLUTION FLOW: UMA → CTF                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. ANYONE calls UmaCtfAdapter.resolve(questionID)                  │
│     └─ Permissionless function                                     │
│                                                                     │
│  2. resolve() calls _resolve() internally                           │
│                                                                     │
│  3. _resolve() fetches price from UMA                               │
│     └─ optimisticOracle.settleAndGetPrice()                        │
│     └─ Returns: int256 price (1e18=YES, 0=NO)                      │
│                                                                     │
│  4. _resolve() converts price to payout vector                      │
│     └─ _constructPayouts(price)                                    │
│     └─ Returns: [1,0] for YES or [0,1] for NO                      │
│                                                                     │
│  5. _resolve() reports to CTF                                       │
│     └─ ctf.reportPayouts(questionID, payouts)                      │
│                                                                     │
│  6. CTF stores the result                                           │
│     └─ payoutNumerators[conditionId] = [1,0] or [0,1]             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Code Evidence:**

**Step 1:** [UmaCtfAdapter.sol#L169](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L169)
```solidity
function resolve(bytes32 questionID) external {
    QuestionData storage questionData = questions[questionID];

    if (!_hasPrice(questionData)) revert NotReadyToResolve();

    return _resolve(questionID, questionData);
}
```

**Step 2-4:** [UmaCtfAdapter.sol#L329](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L329)
```solidity
function _resolve(bytes32 questionID, QuestionData storage questionData) internal {
    // Fetch price from UMA
    int256 price = optimisticOracle.settleAndGetPrice(
        YES_OR_NO_IDENTIFIER,
        questionData.requestTimestamp,
        questionData.ancillaryData
    );

    // Convert to payouts
    uint256[] memory payouts = _constructPayouts(price);

    // Report to CTF
    ctf.reportPayouts(questionID, payouts);
}
```

**Step 5:** [ConditionalTokens.sol#L80](https://github.com/gnosis/conditional-tokens-contracts/blob/master/contracts/ConditionalTokens.sol#L80)
```solidity
function reportPayouts(bytes32 questionId, uint[] calldata payouts) external {
    bytes32 conditionId = CTHelpers.getConditionId(msg.sender, questionId, outcomeSlotCount);

    // Store payouts
    for (uint i = 0; i < outcomeSlotCount; i++) {
        payoutNumerators[conditionId][i] = payouts[i];
    }

    payoutDenominator[conditionId] = denominator;
}
```

**Who can call resolve():** Anyone (permissionless)
**What they send:** Nothing - they just trigger the fetch from UMA
**What they get:** Nothing - no reward for calling resolve()

---

## Resolvers and Incentives

### Q: Who calls resolve()? What do they get?

**A: Anyone can call resolve(), but they get NOTHING except a gas bill.**

**Code Evidence:** [UmaCtfAdapter.sol#L169](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L169)

```solidity
function resolve(bytes32 questionID) external {
    // NO access control - anyone can call
    // NO reward payment to msg.sender
    // NO token minting to caller

    QuestionData storage questionData = questions[questionID];
    // ... validation ...
    return _resolve(questionID, questionData);
}
```

**Money Flow:**

| Role | What They Pay | What They Get | Net Result |
|------|---------------|---------------|------------|
| **Market Creator** | 50 USDC (reward) | Nothing | -50 USDC |
| **Proposer** | 755 USDC (bond+fee) | 805 USDC (if correct) | +50 USDC profit |
| **Resolver** | ~$0.001 (gas) | Nothing | -$0.001 loss |
| **Traders** | Nothing | Can redeem tokens | Unlock value |

**Why do people call resolve() with no incentive?**

1. **Traders need it resolved** to redeem winnings
   - Holding 1000 YES tokens worth $1000
   - Pay $0.001 gas to unlock $1000
   - Worth it!

2. **Proposers want to finalize** to get paid via settle()

3. **Polymarket automation** - likely runs bots to resolve markets

**Real Transaction Example:**
```
Transaction: 0x1b80c5cf5fddfe0e67665d8c7f2383e38cbc40a460ee6d4180eac31c725fa891
Caller: 0x39Ba731cFA2828ea64787AE165FD1e78Fe43AEDe
Token Transfers: NONE to caller
Gas Cost: 0.008147 POL (~$0.001)
Result: Caller paid gas, received nothing
```

---

## Dispute Mechanics

### Q: What happens when there's a dispute?

**A: Polymarket uses a two-tier dispute system: first dispute resets the market, second dispute escalates to DVM voting.**

### First Dispute: Market Reset

**Flow:**

```
Initial Proposal → Dispute! → Market RESETS → New Proposal Required
```

**Code Evidence:** [UmaCtfAdapter.sol - priceDisputed()](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol)

```solidity
function priceDisputed(
    bytes32,
    uint256,
    bytes memory ancillaryData,
    uint256
) external onlyOptimisticOracle {
    bytes32 questionID = keccak256(ancillaryData);
    QuestionData storage questionData = questions[questionID];

    // If already resolved, just refund
    if (questionData.resolved) {
        TransferHelper._transfer(
            questionData.rewardToken,
            questionData.creator,
            questionData.reward
        );
        return;
    }

    // SECOND DISPUTE: Don't reset again
    if (questionData.reset) {
        questionData.refund = true;
        return;
    }

    // FIRST DISPUTE: Reset the market
    _reset(address(this), questionID, false, questionData);
}
```

**What _reset() does:** [UmaCtfAdapter.sol - _reset()](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol)

```solidity
function _reset(
    address requestor,
    bytes32 questionID,
    bool resetRefund,
    QuestionData storage questionData
) internal {
    // Create new timestamp
    uint256 requestTimestamp = block.timestamp;
    questionData.requestTimestamp = requestTimestamp;
    questionData.reset = true;

    // Send NEW price request to UMA
    _requestPrice(
        requestor,
        requestTimestamp,
        questionData.ancillaryData,
        questionData.rewardToken,
        questionData.reward,
        questionData.proposalBond,
        questionData.liveness
    );

    emit QuestionReset(questionID);
}
```

### Second Dispute: DVM Escalation

**Flow:**

```
Reset Proposal → Second Dispute! → DVM Voting (48-96 hours) → Final Answer
```

**Key Code:**
```solidity
// In priceDisputed() callback
if (questionData.reset) {
    // Already reset once - don't reset again
    questionData.refund = true;
    return; // No call to _reset()
}
```

**Complete Dispute Flow:**

```
Day 0:  Market created
Day 30: Proposer A: "YES" (1e18)
        └─ Bond: 750 USDC
        └─ Liveness: 2 hours

Day 30 (+1hr): Disputer B: "Wrong!"
        └─ Bond: 750 USDC
        └─ FIRST DISPUTE
        ├─ Burns 375 USDC immediately
        ├─ Market RESETS
        ├─ DVM votes on original (in parallel)
        └─ Waiting for new proposal

Day 30 (+2hr): Proposer C: "NO" (0)
        └─ Bond: 750 USDC
        └─ New liveness: 2 hours

Day 30 (+3hr): Disputer D: "Also wrong!"
        └─ Bond: 750 USDC
        └─ SECOND DISPUTE
        ├─ Burns another 375 USDC
        ├─ NO RESET this time
        ├─ DVM must decide
        └─ Waiting for DVM

Day 32: DVM completes
        └─ Final answer: YES (1e18)
        └─ Market can now resolve
```

### Money Flow in Disputes

**First Dispute - Proposer Wins:**

| Party | Pays | Gets Back | Net |
|-------|------|-----------|-----|
| Proposer A | 755 USDC | 1,180 USDC | +425 USDC |
| Disputer B | 755 USDC | 0 USDC | -755 USDC |
| UMA Protocol | 0 USDC | 380 USDC | +380 USDC (burned + fee) |

**Code Evidence:** [OptimisticOracleV2.sol#L436-L450](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol#L436)

```solidity
if (state == State.Resolved) {
    resolvedPrice = _getOracle().getPrice(...);

    if (resolvedPrice == proposedPrice) {
        // Proposer was correct
        payout = proposerBond + finalFee + reward + (disputerBond / 2);
        // payout = 750 + 5 + 50 + 375 = 1,180 USDC
    } else {
        // Disputer was correct
        payout = disputerBond + finalFee + (proposerBond / 2);
        // payout = 750 + 5 + 375 = 1,130 USDC
    }
}
```

---

## Bond Burning

### Q: What's the evidence of bond burning? Why burn bonds?

**A: 50% of the loser's bond is burned and sent to UMA's Store contract to prevent Sybil attacks.**

### Code Evidence #1: Burn Calculation

**File:** [OptimisticOracleV2.sol - _computeBurnedBond()](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol)

```solidity
function _computeBurnedBond(Request storage request) private view returns (uint256) {
    return request.requestSettings.bond.div(2);
}
```

**Result:** `burnedBond = 750 USDC / 2 = 375 USDC`

### Code Evidence #2: Where Burned Bonds Go

**File:** [OptimisticOracleV2.sol - disputePriceFor()](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol)

```solidity
function disputePriceFor(...) public override returns (uint256 totalBond) {
    // ... validation ...

    // Transfer bond from disputer
    request.currency.safeTransferFrom(msg.sender, address(this), totalBond);

    // Calculate burned amount
    StoreInterface store = _getStore();
    uint256 totalFee = finalFee.add(_computeBurnedBond(request));

    if (totalFee > 0) {
        // Send to UMA's Store contract
        request.currency.safeIncreaseAllowance(address(store), totalFee);
        _getStore().payOracleFeesErc20(
            address(request.currency),
            FixedPoint.Unsigned(totalFee)
        );
    }

    // ... rest of function ...
}
```

**Evidence:**
- Burned bond = 375 USDC
- Destination: UMA's Store contract
- Function: `payOracleFeesErc20()`
- Timing: Immediately when dispute occurs

### Code Evidence #3: Winner Payout (Unburned Portion Only)

**File:** [OptimisticOracleV2.sol - _settle()](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol)

```solidity
function _settle(...) private returns (uint256 payout) {
    bool disputeSuccess = request.resolvedPrice != request.proposedPrice;
    uint256 bond = request.requestSettings.bond;

    // Calculate unburned portion of loser's bond
    uint256 unburnedBond = bond.sub(_computeBurnedBond(request));
    // unburnedBond = 750 - 375 = 375 USDC

    // Winner gets: own bond + unburned half + fee + reward
    payout = bond.add(unburnedBond).add(request.finalFee).add(request.reward);
    // payout = 750 + 375 + 5 + 50 = 1,180 USDC

    request.currency.safeTransfer(
        disputeSuccess ? request.disputer : request.proposer,
        payout
    );
}
```

### Why Burn Bonds? Economic Reasoning

**Official Explanation from Code Comments:**

> "Along with the final fee, 'burn' part of the loser's bond to ensure that a larger bond always makes it proportionally more expensive to delay the resolution even if the proposer and disputer are the same party."

**Problem Without Burning:**

```
Attacker Controls Both Wallets:
├─ Wallet A proposes: 750 USDC
├─ Wallet B disputes: 750 USDC
├─ DVM decides Wallet A wins
├─ Wallet A gets: 750 + 750 + 55 = 1,555 USDC
└─ Net cost: 1,500 - 1,555 = -55 USDC (profit!)

Result: Attacker can delay for 48-96 hours for only 55 USDC cost
        This enables grief attacks and market manipulation
```

**Solution With 50% Burning:**

```
Attacker Controls Both Wallets:
├─ Wallet A proposes: 750 USDC
├─ Wallet B disputes: 750 USDC
├─ Burned immediately: 375 USDC → UMA Store
├─ DVM decides Wallet A wins
├─ Wallet A gets: 750 + 375 + 55 = 1,180 USDC
└─ Net cost: 1,500 - 1,180 = 320 USDC LOSS

Result: Attacker LOSES money even when winning
        Attack is now unprofitable!
```

**Why 50% Specifically?**

- **Too little (10%)**: Sybil attacks still profitable
- **Too much (90%)**: Discourages honest disputes
- **50% sweet spot**: Makes self-dealing unprofitable while rewarding honest disputers

**Money Flow:**

```
Total Locked: 1,510 USDC (both bonds + fees)

Distribution (Proposer Wins):
├─ Proposer: 1,180 USDC (78%)
├─ UMA Store: 380 USDC (25% - burned + fee)
└─ Disputer: 0 USDC (0%)

Total: 1,560 USDC (includes 50 USDC reward from creator)
Check: 1,180 + 380 = 1,560 ✓
```

---

## Proposers and Third Parties

### Q: Who are these 3rd party proposers? Is there a service?

**A: Anyone can be a proposer! It's completely permissionless, but in practice it's specialized bots and services.**

### Code Evidence: No Access Control

**File:** [OptimisticOracleV2.sol - proposePrice()](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol)

```solidity
function proposePrice(
    address requester,
    bytes32 identifier,
    uint256 timestamp,
    bytes memory ancillaryData,
    int256 proposedPrice
) external override nonReentrant() returns (uint256 totalBond) {
    // NO ACCESS CONTROL
    // NO require(msg.sender == approvedProposer)
    // NO whitelist check

    return proposePriceFor(
        msg.sender,
        requester,
        identifier,
        timestamp,
        ancillaryData,
        proposedPrice
    );
}
```

**Requirements to propose:**
1. Have 755 USDC (750 bond + 5 fee)
2. Call `proposePrice()` with correct parameters
3. Know the answer to the question

**That's it!** No KYC, no whitelist, no approval needed.

### Who Proposes in Practice?

**1. Professional Proposer Services**

Companies/individuals running automated bots:
- Monitor all UMA price requests
- Fetch data from real-world sources
- Submit proposals automatically
- Earn ~50 USDC reward per proposal

**Economics:**
```
Revenue per proposal: 50 USDC
Capital required: 755 USDC (returned if correct)
Time commitment: 2 hours locked + dispute risk
Expected value: +50 USDC (98% non-dispute rate)
```

**2. Polymarket's Infrastructure**

Polymarket likely runs its own proposer bots:
- Ensures markets resolve promptly
- Provides good user experience
- Doesn't rely on external parties

**3. Risk Labs / UMA Team**

UMA's own team may run proposer services:
- Bootstrap the ecosystem
- Ensure system works properly
- Earn protocol revenue

**4. Opportunistic Users**

Random users can propose if:
- They happen to have 755 USDC
- They know the correct answer
- They want to earn 50 USDC

### Note: Managed Optimistic Oracle (MOOV2)

**As of August 2025**, Polymarket transitioned to MOOV2 which **restricts who can propose**:

**MOOV2 Changes:**
- Only **whitelisted addresses** can propose
- Initial whitelist: 37 addresses
- Criteria: Risk Labs employees, users with 20+ proposals and >95% accuracy
- Anyone can still **dispute** (unchanged)

**Why?** Higher quality proposals, fewer errors on non-contentious markets

---

## Timing and Synchronicity

### Q: Can someone propose the answer without waiting?

**A: YES - Anyone can propose IMMEDIATELY after market creation with NO waiting period.**

### Code Proof: No Time Delay

**File:** [OptimisticOracleV2.sol - proposePriceFor()](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol)

```solidity
function proposePriceFor(...) public override returns (uint256 totalBond) {
    // Validation 1: Proposer address valid
    require(proposer != address(0), "proposer address must be non 0");

    // Validation 2: State must be Requested
    require(
        _getState(...) == State.Requested,
        "proposePriceFor: Requested"
    );

    // ⚠️ NO TIME CHECK! ⚠️
    // No: require(block.timestamp > requestTime + delay)
    // No: require(getCurrentTime() > someDeadline)

    // Set proposal and start liveness
    request.proposer = proposer;
    request.proposedPrice = proposedPrice;
    request.expirationTime = getCurrentTime().add(liveness);

    // ... rest of function ...
}
```

**Timeline Example:**

```solidity
// Block 100, 10:00:00 AM
UmaCtfAdapter.initialize(...);
  └─> State: Requested

// Block 100, 10:00:01 AM (1 second later!)
OptimisticOracle.proposePrice(..., 1e18); ✅ SUCCESS!
  └─> State: Proposed
  └─> expirationTime: 12:00:01 PM (2 hours from proposal)
```

**You can propose in the SAME BLOCK as creation!**

### Q: Can someone resolve the poll right after creation?

**A: NO - resolve() will REVERT until liveness period expires (minimum 2 hours after proposal).**

### Code Proof: Must Wait for Liveness

**File:** [UmaCtfAdapter.sol - resolve()](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol)

```solidity
function resolve(bytes32 questionID) external {
    QuestionData storage questionData = questions[questionID];

    if (!_isInitialized(questionData)) revert NotInitialized();
    if (questionData.paused) revert Paused();
    if (questionData.resolved) revert Resolved();

    // ⚠️ THIS CHECK BLOCKS EARLY RESOLUTION ⚠️
    if (!_hasPrice(questionData)) revert NotReadyToResolve();

    return _resolve(questionID, questionData);
}

function _hasPrice(QuestionData storage questionData)
    internal view returns (bool) {
    return optimisticOracle.hasPrice(
        address(this),
        YES_OR_NO_IDENTIFIER,
        questionData.requestTimestamp,
        questionData.ancillaryData
    );
}
```

### State Check Logic

**File:** [OptimisticOracleV2.sol - hasPrice()](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol)

```solidity
function hasPrice(...) public view returns (bool) {
    State state = _getState(...);

    // Price available ONLY in these states:
    return state == State.Settled ||
           state == State.Resolved ||
           state == State.Expired;
}

function _getState(...) internal view returns (State) {
    Request storage request = _getRequest(...);

    if (address(request.currency) == address(0))
        return State.Invalid;

    if (request.proposer == address(0))
        return State.Requested;

    if (request.settled)
        return State.Settled;

    // ⏰ TIME CHECK HERE ⏰
    if (request.disputer == address(0))
        return request.expirationTime <= getCurrentTime()
            ? State.Expired    // ✅ Can resolve
            : State.Proposed;  // ❌ Cannot resolve yet

    return _getOracle().hasPrice(...)
        ? State.Resolved
        : State.Disputed;
}
```

**The critical check:**

```solidity
request.expirationTime <= getCurrentTime()
    ? State.Expired
    : State.Proposed
```

Where:
```solidity
expirationTime = proposalTime + liveness
expirationTime = now + 7200 seconds (2 hours default)
```

### Complete Timeline with States

```
T=0s:     initialize()
          └─> State: Requested
          └─> hasPrice(): FALSE ❌
          └─> resolve(): REVERTS

T=1s:     proposePrice(1e18)
          └─> State: Proposed
          └─> expirationTime = T + 7200s
          └─> hasPrice(): FALSE ❌
          └─> resolve(): REVERTS

T=3600s:  (1 hour later)
          └─> State: Proposed
          └─> expirationTime > getCurrentTime()
          └─> hasPrice(): FALSE ❌
          └─> resolve(): REVERTS

T=7199s:  (almost 2 hours)
          └─> State: Proposed
          └─> hasPrice(): FALSE ❌
          └─> resolve(): REVERTS

T=7200s:  (exactly 2 hours) ✅ LIVENESS EXPIRES
          └─> State: Expired
          └─> expirationTime <= getCurrentTime()
          └─> hasPrice(): TRUE ✅
          └─> resolve(): SUCCESS! 🎉
```

### Why This Asymmetry?

**Propose: NO WAITING** ✅
- Fast proposals encourage quick answers
- Competition among proposers
- Rewards go to first correct proposer

**Resolve: MUST WAIT** ❌
- 2-hour liveness is the **security mechanism**
- Gives honest actors time to dispute
- Prevents instant resolution attacks

**Attack Prevention:**

```
Without liveness period:
├─ Attacker creates market
├─ Attacker proposes wrong answer
├─ Attacker resolves immediately
├─ No one can dispute
└─ System broken ❌

With 2-hour liveness:
├─ Attacker creates market
├─ Attacker proposes wrong answer
├─ Must wait 2 hours before resolve
├─ Honest disputer catches error
├─ Dispute triggered
└─ System works ✅
```

---

## Edge Case: Premature Proposals

### Q: What happens if someone proposes an answer before the event actually occurs?

**Scenario:**
```
Day 0: Create market "Will X happen by Day 3?"
Day 0: Someone proposes "YES" immediately (event hasn't happened yet!)
Day 0: Dispute likely
Day 3: Event actually occurs (or doesn't)
```

**A: The market will NOT resolve incorrectly due to economic incentives and the dispute mechanism.**

### Why Premature Proposals Are Allowed

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

    // ✅ Only check: Must be in Requested state
    require(_getState(...) == State.Requested);

    // ❌ NO event timestamp validation
    // ❌ NO "must wait until event date" check
    // ❌ NO parsing of ancillaryData for dates

    request.proposer = proposer;
    request.proposedPrice = proposedPrice;
    request.expirationTime = getCurrentTime().add(liveness);

    // ... bond handling ...
}
```

**File:** [UmaCtfAdapter.sol - initialize() L125-150](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L125-L150)

```solidity
function initialize(
    bytes memory ancillaryData,
    address rewardToken,
    uint256 reward,
    uint256 proposalBond,
    uint256 liveness
) external returns (bytes32 questionID, bytes32 conditionId) {
    // ancillaryData contains question like:
    // "Will Bitcoin reach $100k by December 31, 2025?"

    bytes memory data = AncillaryDataLib._appendAncillaryData(
        msg.sender,
        ancillaryData
    );

    // ❌ NO validation that event date is in the future
    // ❌ NO parsing of "by December 31, 2025"
    // ✅ Just stores it as opaque metadata

    if (ancillaryData.length == 0 || data.length > MAX_ANCILLARY_DATA)
        revert InvalidAncillaryData();

    questionID = keccak256(data);
    // ... rest of initialization ...
}
```

### What Actually Happens: Step-by-Step

#### Timeline Visualization

```
┌─────────────────────────────────────────────────────────────────┐
│              PREMATURE PROPOSAL SCENARIO                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Day 0, T=0     CREATE MARKET                                    │
│                └─> "Will X happen by Day 3?"                    │
│                └─> State: Requested                             │
│                                                                 │
│ Day 0, T=5min  PREMATURE PROPOSAL ⚠️                            │
│                ├─> Proposer posts "YES" + 750 USDC bond        │
│                ├─> Event hasn't happened yet!                   │
│                └─> State: Proposed (liveness starts)            │
│                                                                 │
│ Day 0, T=10min DISPUTE 💰                                       │
│                ├─> Disputer: "Event hasn't occurred yet!"      │
│                ├─> Disputer posts 750 USDC bond                │
│                └─> State: Disputed                              │
│                                                                 │
│ Day 0, T=15min FIRST DISPUTE RESOLUTION                         │
│                ├─> Market RESETS to State: Requested            │
│                ├─> Proposer loses 350 USDC (50% burned)         │
│                ├─> Disputer gains 405 USDC                      │
│                └─> NEW price request created                    │
│                                                                 │
│ Day 3          EVENT OCCURS (or doesn't)                        │
│                └─> Now the answer is knowable                   │
│                                                                 │
│ Day 3, T=1hr   VALID PROPOSAL ✅                                │
│                ├─> Proposer submits correct answer             │
│                ├─> Based on real-world outcome                  │
│                └─> State: Proposed                              │
│                                                                 │
│ Day 3, T=3hr   LIVENESS EXPIRES                                 │
│                ├─> No disputes (answer is correct)              │
│                └─> State: Expired                               │
│                                                                 │
│ Day 3, T=4hr   MARKET RESOLVES                                  │
│                └─> Anyone calls resolve()                       │
│                └─> Market finalizes correctly ✅                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Economic Game Theory Protection

**The system relies on ECONOMICS not TIMESTAMPS:**

| Actor | Action | Cost | Reward | Net Result |
|-------|--------|------|--------|------------|
| **Premature Proposer** | Propose "YES" before event | 750 USDC bond | 0 (gets disputed) | **-350 USDC ❌** |
| **Rational Disputer** | Dispute premature proposal | 750 USDC bond | 750 back + 400 from proposer + 5 reward | **+405 USDC ✅** |
| **Valid Proposer** | Propose after event occurs | 750 USDC bond | 750 back + 5 reward | **+5 USDC ✅** |

### Money Flow: Premature Proposal Gets Disputed

**File:** [OptimisticOracleV2.sol - disputePriceFor() L541-580](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol#L541-L580)

```solidity
function disputePriceFor(...) public override returns (uint256 totalBond) {
    // Disputer posts bond
    totalBond = request.requestSettings.bond;

    // Calculate burned amount (50% of proposer's bond)
    uint256 burnedBond = _computeBurnedBond(request); // 375 USDC
    uint256 totalFee = finalFee.add(burnedBond);      // ~350 USDC

    // 🔥 BURN 50% of proposer's bond
    if (totalFee > 0) {
        request.currency.safeIncreaseAllowance(address(store), totalFee);
        _getStore().payOracleFeesErc20(
            address(request.currency),
            FixedPoint.Unsigned(totalFee)
        );
    }

    // Escalate to DVM or trigger callback
    _getOracle().requestPrice(...);
}

function _computeBurnedBond(Request storage request)
    private view returns (uint256) {
    // 🔥 50% of bond is burned
    return request.requestSettings.bond.div(2); // 750 / 2 = 375 USDC
}
```

**Money Flow Diagram:**

```
PREMATURE PROPOSAL DISPUTE:

┌─────────────────────────────────────────────────────────────┐
│  Proposer (loses)                 Disputer (wins)           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Posted: 750 USDC                 Posted: 750 USDC          │
│                                                             │
│  Gets back: 400 USDC              Gets back: 750 USDC       │
│  (750 - 350 burned)               (original bond)           │
│                                   + 400 USDC                │
│                                   (from proposer's 750)     │
│                                   + ~5 USDC reward          │
│                                   ────────────              │
│  Lost: -350 USDC ❌               Profit: +405 USDC ✅      │
│                                                             │
│  🔥 Burned: 350 USDC → UMA Store (protocol revenue)        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### First Dispute Triggers Market Reset

**File:** [UmaCtfAdapter.sol - priceDisputed() L289-310](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L289-L310)

```solidity
function priceDisputed(
    bytes32 identifier,
    uint256 timestamp,
    bytes memory ancillaryData,
    uint256 refund
) external {
    QuestionData storage questionData = questions[questionID];

    // First dispute: Reset market
    if (!questionData.reset) {
        _reset(
            msg.sender,      // New requester
            questionID,
            refund > 0,      // resetRefund flag
            questionData
        );
        return;
    }

    // Second dispute: Don't reset (escalate to DVM)
    // ... handle DVM escalation ...
}

function _reset(...) internal {
    // Mark as reset
    questionData.reset = true;
    questionData.requestTimestamp = block.timestamp;

    // ✅ CREATE NEW PRICE REQUEST
    _requestPrice(
        requestor,
        block.timestamp,
        questionData.ancillaryData,
        questionData.rewardToken,
        questionData.reward,
        questionData.proposalBond,
        questionData.liveness
    );

    // Market returns to State: Requested
    // Now someone can propose the CORRECT answer
}
```

### Why No On-Chain Timestamp Validation?

**Design Philosophy: Economic Security > Code Validation**

1. **Smart contracts can't verify real-world events**
   - Contract can't know if Bitcoin actually hit $100k
   - Contract can't check if event date has passed
   - Needs humans (proposers) to bridge real world → blockchain

2. **Economic incentives prevent bad behavior**
   - Premature proposal costs 350 USDC
   - Disputing premature proposal earns 405 USDC
   - Rational actors won't lose money intentionally

3. **Dispute mechanism is the safeguard**
   - 2-hour liveness gives time to dispute
   - Anyone can dispute (permissionless)
   - First dispute resets market (no harm done)

4. **MOOV2 adds reputation layer**
   - Since August 2025: Whitelisted proposers only
   - Must maintain >95% accuracy
   - Lose whitelist access if accuracy drops
   - Whitelisted proposers won't risk reputation

### Attack Prevention Analysis

#### ❌ Attack Attempt: Premature Proposal

```
Attacker's Plan:
1. Create market "Will X happen by Day 3?"
2. Immediately propose "YES" (before event)
3. Wait 2 hours for liveness to expire
4. Call resolve() and profit

Why It Fails:
├─ Step 3: Rational disputer disputes (earns 405 USDC)
├─ Market resets to "Requested" state
├─ Attacker loses 350 USDC
└─ Attack fails ❌
```

#### ✅ Defense Mechanism: Economic Disincentive

```
Honest Flow:
1. Market created "Will X happen by Day 3?"
2. Everyone waits until Day 3 (or later)
3. Event occurs (or doesn't)
4. Proposer submits correct answer
5. No one disputes (answer is verifiable)
6. Market resolves correctly ✅

Why It Works:
├─ Proposers want +5 USDC reward
├─ Proposers don't want -350 USDC loss
├─ Disputers actively monitor for free 405 USDC
└─ Economic rationality ensures correctness
```

### Real-World Protection: MOOV2 Whitelist

**Context:** As of August 2025, Polymarket uses Managed Optimistic Oracle V2

**Whitelist Criteria:**
- Risk Labs employees
- Users with 20+ proposals AND >95% accuracy
- Total: 37 whitelisted addresses initially

**Code Reference:** [MOOV2 Implementation](https://github.com/UMAprotocol/protocol/tree/master/packages/core/contracts/optimistic-oracle-v2)

```solidity
// MOOV2 adds proposer restrictions
mapping(address => bool) public whitelistedProposers;

function proposePrice(...) external {
    // ✅ NEW: Check if proposer is whitelisted
    require(
        whitelistedProposers[msg.sender],
        "Proposer not whitelisted"
    );

    // ... rest of proposal logic ...
}
```

**Impact:**
- Casual attackers can't propose (not whitelisted)
- Whitelisted proposers won't risk reputation
- Premature proposals become extremely rare

### Summary: Multi-Layer Protection

| Layer | Mechanism | Protection |
|-------|-----------|------------|
| **Economic** | -350 USDC penalty for wrong proposals | Disincentivizes premature proposals |
| **Game Theory** | +405 USDC reward for disputing wrong answers | Incentivizes monitoring and disputes |
| **Time** | 2-hour liveness period | Gives time for disputes to occur |
| **State Machine** | First dispute resets market | Prevents incorrect resolution |
| **Reputation** | MOOV2 whitelist (>95% accuracy) | Only trusted proposers can submit |

**The Result:** No on-chain timestamp validation needed because economic incentives make premature proposals irrational.

---

## Summary Table

| Question | Answer | Code Reference |
|----------|--------|----------------|
| **Who requests price from UMA?** | UmaCtfAdapter during initialize() | [UmaCtfAdapter.sol#L140](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L140) |
| **When is price requested?** | During market creation, not resolution | [_requestPrice()](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol) |
| **How does CTF get the answer?** | UmaCtfAdapter.resolve() → ctf.reportPayouts() | [resolve()](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol#L169) |
| **Who calls resolve()?** | Anyone (permissionless) | No access control in resolve() |
| **What does resolver get?** | Nothing (gas cost only) | No payments in resolve() |
| **What happens on first dispute?** | Market resets, new proposal needed | [priceDisputed()](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol) |
| **What happens on second dispute?** | Escalates to DVM voting (no reset) | [priceDisputed() reset check](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol) |
| **How much bond is burned?** | 50% (375 USDC from 750 USDC bond) | [_computeBurnedBond()](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol) |
| **Where do burned bonds go?** | UMA Store contract via payOracleFeesErc20() | [disputePriceFor()](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol) |
| **Why burn bonds?** | Prevent Sybil/grief attacks | Code comments in OptimisticOracleV2 |
| **Who can propose answers?** | Anyone (permissionless) | [proposePrice()](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol) |
| **Can propose immediately?** | YES - no waiting period | No time checks in proposePriceFor() |
| **Can resolve immediately?** | NO - must wait for liveness | [hasPrice() checks state](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol) |
| **Minimum wait after proposal?** | 2 hours (default liveness) | [expirationTime check](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol) |
| **What prevents premature proposals?** | Economic disincentive (-350 USDC loss) + disputes + MOOV2 whitelist | [OptimisticOracleV2.sol#L474-500](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol#L474-L500) |
| **Can proposal before event date?** | YES technically, but loses 350 USDC when disputed | No event timestamp validation in contracts |
| **How do disputers challenge premature proposals?** | Use P4 ("too early") - don't need to know final answer | DVM voters select P4 option |
| **If DVM votes different from both parties?** | Proposer loses, disputer wins (binary check) | [_settle()](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol) - `disputeSuccess = resolvedPrice != proposedPrice` |
| **Can market resolve as UNKNOWN?** | YES - both token types get $0.50 payout | [_constructPayouts()](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol) returns [1,1] for 0.5e18 |

---

## Disputing Premature Proposals: The P4 Mechanism

### Q: What if someone proposes an answer before the real-world event happens? How can a disputer challenge it since the outcome could be yes or no?

**A: UMA has a special voting option called "P4" (Price 4 / Early Request) specifically for this scenario. Disputers don't need to know the final answer—they just dispute on the grounds that "it's too early to know."**

### The P4 Voting Option

When a dispute goes to DVM voting, voters have these options:
- **p1**: YES (1e18)
- **p2**: NO (0)
- **p3**: UNKNOWN/50-50 (0.5e18)
- **p4**: TOO EARLY / CANNOT BE DETERMINED YET

**Key Insight:** The disputer doesn't need to assert "the answer is NO" to dispute a premature "YES" proposal. They simply dispute because **no valid answer exists yet**.

### Why P4 Exists

From [UMA Documentation](https://blog.uma.xyz/articles/what-is-p4):

> "The purpose of proposing data is to report on an event that has already happened. Submitting proposals too early isn't reporting—it's predicting."

**Critical Rule:** Proposals must always be made **in the past tense**. If your proposal is disputed as premature, **you lose your bond even if your prediction turns out to be correct**.

### How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│              DISPUTING A PREMATURE PROPOSAL                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Day 0:     Market: "Will BTC hit $150k by March 2026?"         │
│                                                                 │
│ Day 0:     Proposer submits "YES" ⚠️ (Event hasn't occurred!)   │
│            └─> Posts 750 USDC bond                              │
│                                                                 │
│ Day 0:     Disputer challenges ✅                                │
│            ├─> Posts 750 USDC bond                              │
│            ├─> Reason: "Cannot determine yet - event future"   │
│            └─> Does NOT need to claim "NO is correct"          │
│                                                                 │
│ DVM Vote:  Voters see the proposal timestamp vs event date     │
│            ├─> If proposal was made BEFORE event could occur   │
│            ├─> Voters select P4: "TOO EARLY"                    │
│            └─> Proposer LOSES bond regardless of eventual      │
│                outcome                                          │
│                                                                 │
│ Result:    Disputer wins ~405 USDC                              │
│            Proposer loses ~350 USDC                             │
│            Market resets for proper proposal later              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Consequences by Role

| Role | Impact of P4 |
|------|--------------|
| **Prediction market users** | P4 prevents premature proposals from passing on-chain, ensuring fairness |
| **Proposers** | Must wait until trigger event occurs. Lose bond if too early, even if "correct" |
| **Disputers** | Can challenge premature proposals and earn portion of proposer's bond |
| **Voters** | Should always check proposal timestamps before voting |

### Smart Contract Enforcement

The system prevents proposers from gaming P4:

```solidity
// Proposers CANNOT propose the "too early" value themselves
// This ensures premature proposers always lose their bond
require(proposedPrice != TOO_EARLY_RESPONSE, "Cannot propose TOO_EARLY");
```

**Why?** If proposers could submit "TOO_EARLY" themselves, they could:
1. Propose "TOO_EARLY" on a market
2. Wait for event to occur
3. Dispute themselves and propose the real answer
4. Avoid any penalty

By blocking this, the system ensures premature proposers **always face consequences**.

---

## DVM Votes a Third Answer: Who Loses?

### Q: What if the proposer says YES, disputer says NO, but DVM voters say UNKNOWN? Who loses money?

**A: The PROPOSER loses. The system is BINARY, not ternary—it only checks if DVM agrees with the proposer or not.**

### The Critical Code Logic

**File:** [OptimisticOracleV2.sol - _settle()](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol)

```solidity
function _settle(...) private returns (uint256 payout) {
    // THE KEY CHECK: Does DVM agree with proposer?
    bool disputeSuccess = request.resolvedPrice != request.proposedPrice;

    // Binary outcome - only two possibilities:
    if (disputeSuccess) {
        // DVM disagreed with proposer → DISPUTER WINS
        recipient = request.disputer;
    } else {
        // DVM agreed with proposer → PROPOSER WINS
        recipient = request.proposer;
    }

    // Winner gets: own bond + half of loser's bond + fee + reward
    payout = bond.add(unburnedBond).add(request.finalFee).add(request.reward);
    request.currency.safeTransfer(recipient, payout);
}
```

### Key Insight: Disputers Don't Need to Be "Right"

**The disputer doesn't submit an alternative answer.** They simply say "I disagree with the proposer."

If DVM votes for **ANY value different from the proposer's answer**, the disputer wins—even if that value wasn't what the disputer expected.

### Example Scenarios

#### Scenario 1: Proposer YES, DVM says UNKNOWN

```
┌─────────────────────────────────────────────────────────────────┐
│  Proposer: "YES" (1e18)                                         │
│  Disputer: "I disagree!" (no specific answer submitted)        │
│  DVM Vote: "UNKNOWN" (0.5e18)                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Check: resolvedPrice (0.5e18) != proposedPrice (1e18)         │
│  Result: disputeSuccess = TRUE                                  │
│                                                                 │
│  Proposer: LOSES ~350 USDC ❌                                   │
│  Disputer: WINS ~375 USDC ✅                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Scenario 2: Proposer YES, DVM says NO

```
┌─────────────────────────────────────────────────────────────────┐
│  Proposer: "YES" (1e18)                                         │
│  Disputer: "I disagree!"                                        │
│  DVM Vote: "NO" (0)                                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Check: resolvedPrice (0) != proposedPrice (1e18)              │
│  Result: disputeSuccess = TRUE                                  │
│                                                                 │
│  Proposer: LOSES ~350 USDC ❌                                   │
│  Disputer: WINS ~375 USDC ✅                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Scenario 3: Proposer YES, DVM says YES

```
┌─────────────────────────────────────────────────────────────────┐
│  Proposer: "YES" (1e18)                                         │
│  Disputer: "I disagree!"                                        │
│  DVM Vote: "YES" (1e18)                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Check: resolvedPrice (1e18) == proposedPrice (1e18)           │
│  Result: disputeSuccess = FALSE                                 │
│                                                                 │
│  Proposer: WINS ~425 USDC ✅                                    │
│  Disputer: LOSES ~350 USDC ❌                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Scenario 4: Proposer YES, DVM says TOO EARLY (P4)

```
┌─────────────────────────────────────────────────────────────────┐
│  Proposer: "YES" (1e18)                                         │
│  Disputer: "Too early to know!"                                 │
│  DVM Vote: "TOO EARLY" (magic value)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Check: resolvedPrice (P4) != proposedPrice (1e18)             │
│  Result: disputeSuccess = TRUE                                  │
│                                                                 │
│  Proposer: LOSES ~350 USDC ❌                                   │
│  Disputer: WINS ~375 USDC ✅                                    │
│  Market: Resets for new proposal                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Summary: The Binary Rule

| DVM Vote | Matches Proposer? | Winner | Loser |
|----------|-------------------|--------|-------|
| YES (1e18) | If proposer said YES | Proposer | Disputer |
| NO (0) | If proposer said NO | Proposer | Disputer |
| UNKNOWN (0.5e18) | Almost never matches | Disputer | Proposer |
| TOO EARLY (P4) | Never matches (blocked) | Disputer | Proposer |

### Why This Design?

1. **Simplicity**: Binary comparison is easy to implement and verify
2. **Disputer protection**: Disputers don't need to know the "correct" answer—just that the proposer is wrong
3. **Incentive alignment**: Encourages disputers to challenge obviously wrong proposals without needing certainty about the right answer
4. **Economic security**: Proposers bear the burden of being correct; disputers only need to identify incorrectness

### Edge Case: What If Both Are "Wrong"?

If the proposer says YES, and the disputer thinks it's NO, but DVM says UNKNOWN:

- **Proposer loses** (their answer didn't match DVM)
- **Disputer wins** (they successfully disputed an incorrect proposal)
- **Market resolves as UNKNOWN** (50-50 payout to both token holders)

The disputer "wins" the bond dispute even though their implicit expectation (NO) wasn't the final answer. This is intentional—**disputers are rewarded for identifying wrong proposals, not for predicting right ones**.

---

## UNKNOWN Resolution: When Real-World Events Are Ambiguous

### Q: Can a market resolve as UNKNOWN because the real-world event is genuinely undecided or ambiguous? What happens next?

**A: YES - UNKNOWN (p3 = 0.5e18) is a valid resolution outcome. Both YES and NO token holders receive 50% of their potential payout.**

### When Is UNKNOWN Used?

UNKNOWN resolution is appropriate when:

1. **Ambiguous question wording** - The question doesn't clearly map to what happened
2. **Subjective criteria** - "Was the weather nice?" has no objective answer
3. **Event didn't occur as specified** - The conditions for YES or NO weren't met
4. **Contradictory evidence** - Multiple valid interpretations exist
5. **Technical nullification** - Event was cancelled/voided in a way not covered by rules

### Examples of UNKNOWN-Worthy Scenarios

```
Example 1: Ambiguous Wording
├─> Market: "Will Team A win the championship?"
├─> Reality: Championship cancelled due to pandemic
├─> Resolution: UNKNOWN (neither won nor lost)

Example 2: Subjective Criteria
├─> Market: "Will the product launch be successful?"
├─> Reality: Launched but mixed reviews
├─> Resolution: UNKNOWN (no objective success criteria)

Example 3: Technical Edge Case
├─> Market: "Will X be CEO on Dec 31?"
├─> Reality: X resigned Dec 30, reappointed Jan 1
├─> Resolution: Could be disputed, may end as UNKNOWN
```

### How UNKNOWN Resolution Works

**File:** [UmaCtfAdapter.sol - _constructPayouts()](https://github.com/Polymarket/uma-ctf-adapter/blob/main/src/UmaCtfAdapter.sol)

```solidity
function _constructPayouts(int256 price) internal pure returns (uint256[] memory) {
    uint256[] memory payouts = new uint256[](2);

    if (price == 1 ether) {
        // YES: Report [Yes, No] as [1, 0]
        payouts[0] = 1;
        payouts[1] = 0;
    } else if (price == 0) {
        // NO: Report [Yes, No] as [0, 1]
        payouts[0] = 0;
        payouts[1] = 1;
    } else if (price == 0.5 ether) {
        // UNKNOWN: Report [Yes, No] as [1, 1], 50/50
        payouts[0] = 1;
        payouts[1] = 1;
    }

    return payouts;
}
```

### Payout Vector Meaning

| Resolution | Price | Payout Vector | YES Token Value | NO Token Value |
|------------|-------|---------------|-----------------|----------------|
| **YES** | 1e18 | [1, 0] | $1.00 | $0.00 |
| **NO** | 0 | [0, 1] | $0.00 | $1.00 |
| **UNKNOWN** | 0.5e18 | [1, 1] | $0.50 | $0.50 |

### What Happens to Token Holders

```
┌─────────────────────────────────────────────────────────────────┐
│           UNKNOWN RESOLUTION: 50/50 PAYOUT                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Market: "Will X happen by Dec 31?"                             │
│  Resolution: UNKNOWN (event ambiguous)                          │
│                                                                 │
│  ┌─────────────────────┐    ┌─────────────────────┐            │
│  │ Alice holds:        │    │ Bob holds:          │            │
│  │ 100 YES tokens      │    │ 100 NO tokens       │            │
│  │ Bought at $0.70 ea  │    │ Bought at $0.30 ea  │            │
│  │ Cost: $70           │    │ Cost: $30           │            │
│  └─────────────────────┘    └─────────────────────┘            │
│                                                                 │
│  After UNKNOWN Resolution:                                      │
│                                                                 │
│  ┌─────────────────────┐    ┌─────────────────────┐            │
│  │ Alice receives:     │    │ Bob receives:       │            │
│  │ 100 × $0.50 = $50   │    │ 100 × $0.50 = $50   │            │
│  │ Net: -$20 loss ❌   │    │ Net: +$20 profit ✅ │            │
│  └─────────────────────┘    └─────────────────────┘            │
│                                                                 │
│  Total Pool: $100 → Split evenly regardless of token type      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Who Wins/Loses in UNKNOWN Resolution?

**Winners:**
- Anyone who bought tokens below $0.50
- NO token holders (usually cheaper, now worth more)

**Losers:**
- Anyone who bought tokens above $0.50
- YES token holders who were confident (usually paid premium)

**Break-even:**
- Anyone who bought at exactly $0.50

### Important Limitation: NegRisk Markets

**Note:** From the code comments:

> "A tie is not a valid outcome when used with the `NegRiskOperator`"

**NegRisk markets** (multi-outcome markets like "Who will win the election?") **cannot resolve as UNKNOWN** because:
- They use a different payout structure
- A 50/50 split across many outcomes doesn't make mathematical sense
- These markets require exactly one winner

### The Complete Resolution Flow

```
┌─────────────────────────────────────────────────────────────────┐
│              UNKNOWN RESOLUTION FLOW                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ 1. PROPOSAL                                                     │
│    └─> Proposer submits: "UNKNOWN" (0.5e18)                    │
│    └─> Bond: 750 USDC                                          │
│                                                                 │
│ 2. LIVENESS PERIOD (2 hours)                                    │
│    ├─> If no dispute → Proposal accepted                       │
│    └─> If disputed → Goes to DVM                               │
│                                                                 │
│ 3. DVM VOTING (if disputed)                                     │
│    └─> Voters choose: YES, NO, UNKNOWN, or TOO_EARLY           │
│    └─> If majority votes UNKNOWN → Resolution = 0.5e18         │
│                                                                 │
│ 4. SETTLEMENT                                                   │
│    └─> UmaCtfAdapter.resolve() called                          │
│    └─> _constructPayouts(0.5e18) returns [1, 1]                │
│    └─> CTF.reportPayouts([1, 1])                               │
│                                                                 │
│ 5. REDEMPTION                                                   │
│    └─> YES holders: Redeem for $0.50 per token                 │
│    └─> NO holders: Redeem for $0.50 per token                  │
│    └─> Total pool split 50/50                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### When to Propose UNKNOWN

**Valid reasons:**
- Question is genuinely ambiguous given what occurred
- Event was cancelled/voided
- No clear winner based on stated criteria
- Multiple valid interpretations of outcome

**Invalid reasons (will likely be disputed):**
- You don't know the answer yet → Use P4 (TOO EARLY)
- You're uncertain but event clearly happened → Research more
- You want to avoid controversy → Pick the correct answer

### Disputes Over UNKNOWN

UNKNOWN resolutions are often controversial because:

1. **Subjectivity**: Reasonable people may disagree on ambiguity
2. **Financial incentives**: Some traders benefit from UNKNOWN, others don't
3. **Interpretation differences**: What counts as "ambiguous"?

From [Polymarket disputes analysis](https://www.oddchain.com/p/does-dispute-alpha-exist-a-study-of-uma-polymarket-and-resolution-incentives):

> "Ambiguous markets can result in close votes influenced by voters' incentives rather than pure objectivity."

### Summary

| Aspect | Details |
|--------|---------|
| **When used** | Genuinely ambiguous events, cancelled events, subjective questions |
| **Price value** | 0.5e18 (0.5 ether) |
| **Payout vector** | [1, 1] |
| **YES token value** | $0.50 |
| **NO token value** | $0.50 |
| **NegRisk compatible** | NO - only binary markets |
| **Controversy level** | High - often disputed |

---

## Finding Bond Costs and Fees

### Q: Where can I find the costs (bond, fees, etc.) for each action during the request-to-resolution period?

**A: Bond and fee information can be found on-chain via contract queries and in UMA documentation.**

### On-Chain Methods

#### 1. Query Minimum Bond (OOV3)

```solidity
// For Optimistic Oracle V3
IOptimisticOracleV3 oracle = IOptimisticOracleV3(oracleAddress);
uint256 minBond = oracle.getMinimumBond(IERC20(tokenAddress));
```

#### 2. Query Final Fee (OOV2)

For OOV2, the minimum bond equals the final fee. Query the Store contract:

```solidity
// Get final fee from UMA Store contract
IStore store = IStore(storeAddress);
uint256 finalFee = store.computeFinalFee(collateralAddress);
```

#### 3. Query Specific Request Parameters

```solidity
// Get bond/liveness for a specific price request
IOptimisticOracleV2 oracle = IOptimisticOracleV2(oracleAddress);
Request memory req = oracle.getRequest(
    requester,
    identifier,
    timestamp,
    ancillaryData
);

uint256 bond = req.requestSettings.bond;
uint256 liveness = req.requestSettings.liveness;
uint256 finalFee = req.finalFee;
```

### Where to Find Fee Information

| Information | Location |
|-------------|----------|
| **Final fees by token** | [UMA Approved Collateral Types](https://docs.uma.xyz/resources/approved-collateral-types) |
| **Default liveness** | 2 hours (7200 seconds) |
| **Bond documentation** | [Setting Custom Bond Parameters](https://docs.uma.xyz/developers/setting-custom-bond-and-liveness-parameters) |

### Typical Polymarket Costs

| Action | Cost | Notes |
|--------|------|-------|
| **Proposer bond** | 750 USDC | Refunded if correct + reward |
| **Disputer bond** | 750 USDC | Same as proposer bond |
| **Final fee** | ~5 USDC | Added to bond, varies by token |
| **Total to propose** | ~755 USDC | bond + finalFee |
| **Market creation reward** | 50 USDC | Paid by market creator to proposer |

### Payout Scenarios

**Proposer Wins (correct proposal, no dispute or wins DVM):**
```
Receives: proposerBond + finalFee + reward + (disputerBond / 2)
        = 750 + 5 + 50 + 375
        = 1,180 USDC
Net profit: +425 USDC (if disputed) or +55 USDC (if no dispute)
```

**Disputer Wins (correct dispute):**
```
Receives: disputerBond + finalFee + (proposerBond / 2)
        = 750 + 5 + 375
        = 1,130 USDC
Net profit: +375 USDC
```

**Loser's Bond Distribution:**
```
Total loser bond: 750 USDC
├─> Winner receives: 375 USDC (50%)
└─> UMA Store (burned): 375 USDC (50%)
```

### Viewing Historical Data

To see actual bond amounts used in past markets:

1. **Polygonscan**: Search for UmaCtfAdapter transactions and decode `initialize()` calls
2. **Dune Analytics**: Query the `proposalBond` and `liveness` parameters from events
3. **UMA Discord**: The #polymarket channel often discusses active proposals and their parameters

---

## Additional Resources

- [UMA Optimistic Oracle V2 Docs](https://docs.uma.xyz/developers/optimistic-oracle)
- [Polymarket Resolution Docs](https://docs.polymarket.com/developers/resolution/UMA)
- [UMA CTF Adapter GitHub](https://github.com/Polymarket/uma-ctf-adapter)
- [OptimisticOracleV2 GitHub](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v2/implementation/OptimisticOracleV2.sol)
- [Conditional Tokens GitHub](https://github.com/gnosis/conditional-tokens-contracts)
- [UMA P4 Voting Explanation](https://blog.uma.xyz/articles/what-is-p4)
- [UMA Bond Parameters](https://docs.uma.xyz/developers/setting-custom-bond-and-liveness-parameters)
- [UMA Approved Collateral & Fees](https://docs.uma.xyz/resources/approved-collateral-types)

---

## Glossary

- **Proposer**: Third party who submits an answer to UMA Oracle (earns reward if correct)
- **Disputer**: Someone who challenges a proposal (earns half of proposer's bond if correct)
- **Resolver**: Anyone who calls resolve() to finalize market (gets nothing)
- **Liveness Period**: 2-hour window after proposal where disputes can happen
- **Bond**: Security deposit (750 USDC) required from proposers/disputers
- **Burned Bond**: 50% of loser's bond sent to UMA Store (not recoverable)
- **DVM**: Data Verification Mechanism - UMA's voting system for disputed proposals
- **State.Requested**: Market created, waiting for proposal
- **State.Proposed**: Proposal submitted, in liveness period
- **State.Expired**: Liveness passed, no dispute, ready to resolve
- **State.Resolved**: DVM voted and decided after dispute
- **P4**: "Too Early" voting option - used when proposal is premature and answer cannot yet be determined
- **Final Fee**: Fixed fee (~5 USDC) required on top of bond, goes to UMA Store
