# Polymarket CLOB & Exchange

> Hybrid-decentralized order book for prediction market trading

**Last Updated:** 2026-01-19

## Overview

Polymarket uses a Central Limit Order Book (CLOB) with a hybrid-decentralized architecture:
- **Off-chain:** Order matching and book management by operator
- **On-chain:** Non-custodial settlement via Exchange contract

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      CLOB ARCHITECTURE                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│    TRADERS                          OPERATOR                            │
│   ┌─────────┐                     ┌─────────────┐                       │
│   │ Sign    │────EIP-712 Order───▶│ Order Book  │                       │
│   │ Order   │                     │ (Off-chain) │                       │
│   └─────────┘                     └──────┬──────┘                       │
│                                          │                              │
│                                          │ Match Orders                 │
│                                          ▼                              │
│                                   ┌─────────────┐                       │
│                                   │  Exchange   │◀── On-chain           │
│                                   │  Contract   │    Settlement         │
│                                   └──────┬──────┘                       │
│                                          │                              │
│                        ┌─────────────────┼─────────────────┐            │
│                        ▼                 ▼                 ▼            │
│                   ┌─────────┐      ┌─────────┐      ┌─────────┐        │
│                   │ NORMAL  │      │  MINT   │      │  MERGE  │        │
│                   │ (Swap)  │      │ (Create)│      │ (Burn)  │        │
│                   └─────────┘      └─────────┘      └─────────┘        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Contract Addresses

| Contract | Address | Purpose |
|----------|---------|---------|
| **CTF Exchange** | `0x4bFb41d5B3570DeFd03C39A9A4D8dE6Bd8B8982E` | Binary market trading |
| **NegRisk CTF Exchange** | `0xC5d563A36AE78145C45a50134d48A1215220F80a` | Multi-outcome markets |

---

## Order Structure

Orders are EIP-712 signed structured data.

**Source:** [CTFExchange.sol - Order struct](https://github.com/Polymarket/ctf-exchange/blob/main/src/exchange/CTFExchange.sol)

```solidity
struct Order {
    uint256 salt;           // Random nonce for uniqueness
    address maker;          // Order creator
    address signer;         // Signing key (may differ from maker)
    address taker;          // Specific taker or 0x0 for any
    uint256 tokenId;        // ERC1155 position ID
    uint256 makerAmount;    // Amount maker offers
    uint256 takerAmount;    // Amount maker wants
    uint256 expiration;     // Unix timestamp expiry
    uint256 nonce;          // For bulk cancellation
    uint256 feeRateBps;     // Fee rate in basis points
    uint8 side;             // 0 = BUY, 1 = SELL
    uint8 signatureType;    // Signature scheme
}
```

### Order Signing

```typescript
const domain = {
    name: "Polymarket CTF Exchange",
    version: "1",
    chainId: 137,
    verifyingContract: "0x4bFb41d5B3570DeFd03C39A9A4D8dE6Bd8B8982E"
};

const types = {
    Order: [
        { name: "salt", type: "uint256" },
        { name: "maker", type: "address" },
        { name: "signer", type: "address" },
        { name: "taker", type: "address" },
        { name: "tokenId", type: "uint256" },
        { name: "makerAmount", type: "uint256" },
        { name: "takerAmount", type: "uint256" },
        { name: "expiration", type: "uint256" },
        { name: "nonce", type: "uint256" },
        { name: "feeRateBps", type: "uint256" },
        { name: "side", type: "uint8" },
        { name: "signatureType", type: "uint8" }
    ]
};

const signature = await wallet._signTypedData(domain, types, order);
```

---

## Matching Modes

### NORMAL Mode

Direct token-for-collateral swap with no minting or burning:

```
UserA: BUY 100 YES @ 0.60 USDC
UserB: SELL 100 YES @ 0.60 USDC

Settlement:
  UserA: -60 USDC, +100 YES
  UserB: +60 USDC, -100 YES
```

### MINT Mode

Creates new tokens when complementary orders match:

```
UserA: BUY 100 YES @ 0.60 USDC
UserB: BUY 100 NO @ 0.40 USDC

Total collateral: 0.60 + 0.40 = 1.00 USDC per share

Settlement:
  1. Contract receives 100 USDC total
  2. Contract calls CTF.splitPosition(100 USDC)
  3. Mints 100 YES + 100 NO
  4. UserA receives 100 YES
  5. UserB receives 100 NO
```

**Key insight:** Orders for opposing outcomes can cross if prices sum to 1.00 or more.

### MERGE Mode

Burns tokens when complementary sell orders match:

```
UserA: SELL 100 YES @ 0.60 USDC
UserB: SELL 100 NO @ 0.40 USDC

Settlement:
  1. Contract receives 100 YES from UserA
  2. Contract receives 100 NO from UserB
  3. Contract calls CTF.mergePositions(100)
  4. Contract receives 100 USDC
  5. UserA receives 60 USDC
  6. UserB receives 40 USDC
```

---

## Fee Structure

### Current Rates (as of 2026-01)

| Tier | Maker Fee | Taker Fee |
|------|-----------|-----------|
| Default | 0% | 0% |

Fees may vary; check current rates on Polymarket.

### Fee Calculation

Fees are symmetric based on the minimum of price and (1-price):

```
feeQuote = baseRate × min(price, 1-price) × size
```

**Example:**
- Buying YES at 0.70 (or NO at 0.30)
- min(0.70, 0.30) = 0.30
- feeQuote = baseRate × 0.30 × size

This ensures equal fees regardless of which side you trade.

---

## Exchange Contract Functions

**Source:** [CTFExchange.sol](https://github.com/Polymarket/ctf-exchange/blob/main/src/exchange/CTFExchange.sol)

### fillOrder

Execute a single order.

**Source:** [Trading.sol#L38-L60](https://github.com/Polymarket/ctf-exchange/blob/main/src/exchange/mixins/Trading.sol#L38-L60)

```solidity
function fillOrder(
    Order memory order,
    bytes memory signature,
    uint256 fillAmount
) external returns (uint256 takerAmountFilled);
```

### fillOrders

Execute multiple orders atomically.

**Source:** [Trading.sol#L62-L92](https://github.com/Polymarket/ctf-exchange/blob/main/src/exchange/mixins/Trading.sol#L62-L92)

```solidity
function fillOrders(
    Order[] memory orders,
    bytes[] memory signatures,
    uint256[] memory fillAmounts
) external returns (uint256[] memory takerAmountsFilled);
```

### matchOrders

Match a maker order with taker orders.

**Source:** [Trading.sol#L94-L150](https://github.com/Polymarket/ctf-exchange/blob/main/src/exchange/mixins/Trading.sol#L94-L150)

```solidity
function matchOrders(
    Order memory takerOrder,
    Order[] memory makerOrders,
    bytes memory takerSignature,
    bytes[] memory makerSignatures,
    uint256[] memory takerFillAmounts,
    uint256[] memory makerFillAmounts
) external;
```

### cancelOrder

Cancel a specific order.

**Source:** [NonceManager.sol](https://github.com/Polymarket/ctf-exchange/blob/main/src/exchange/mixins/NonceManager.sol)

```solidity
function cancelOrder(Order memory order) external;
```

### cancelOrders

Cancel multiple orders.

```solidity
function cancelOrders(Order[] memory orders) external;
```

### incrementNonce

Invalidate all orders with current nonce.

```solidity
function incrementNonce() external;
```

---

## Events

```solidity
event OrderFilled(
    bytes32 indexed orderHash,
    address indexed maker,
    address indexed taker,
    uint256 makerAssetId,
    uint256 takerAssetId,
    uint256 makerAmountFilled,
    uint256 takerAmountFilled,
    uint256 fee
);

event OrdersMatched(
    bytes32 indexed takerOrderHash,
    address indexed takerOrderMaker,
    uint256 makerAssetId,
    uint256 takerAssetId,
    uint256 makerAmountFilled,
    uint256 takerAmountFilled
);

event OrderCancelled(bytes32 indexed orderHash);

event NonceIncremented(address indexed maker, uint256 newNonce);

event TokenRegistered(
    uint256 indexed token0,
    uint256 indexed token1,
    bytes32 indexed conditionId
);
```

---

## API Endpoints

### REST API

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/markets` | GET | List all markets |
| `/markets/{id}` | GET | Get market details |
| `/book/{tokenId}` | GET | Get order book for token |
| `/orders` | POST | Submit new order |
| `/orders/{id}` | DELETE | Cancel order |
| `/trades` | GET | Get trade history |

### WebSocket

```javascript
// Subscribe to order book updates
ws.send(JSON.stringify({
    type: "subscribe",
    channel: "book",
    market: "0x..."  // conditionId
}));

// Subscribe to trades
ws.send(JSON.stringify({
    type: "subscribe",
    channel: "trades",
    market: "0x..."
}));
```

---

## Price Mechanics

### Binary Market Pricing

For binary markets (YES/NO):
- YES price + NO price should equal ~1.00 USDC
- Arbitrage keeps prices in sync

```
If YES trades at $0.65
Then NO should trade at ~$0.35

Total: $0.65 + $0.35 = $1.00
```

### Implied Probability

Token price represents implied probability of outcome:
- YES at $0.65 = 65% implied probability of YES
- NO at $0.35 = 35% implied probability of YES

---

## Trading Flow

### 1. Prepare for Trading

```solidity
// Approve USDC for CTF (for splitting)
USDC.approve(CTF_ADDRESS, type(uint256).max);

// Approve CTF tokens for Exchange
CTF.setApprovalForAll(EXCHANGE_ADDRESS, true);
```

### 2. Get Outcome Tokens (if needed)

```solidity
// Split USDC into YES + NO tokens
CTF.splitPosition(
    USDC,
    bytes32(0),
    conditionId,
    [1, 2],
    amount
);
```

### 3. Create and Sign Order

```typescript
const order = {
    salt: randomBigInt(),
    maker: wallet.address,
    signer: wallet.address,
    taker: ethers.constants.AddressZero,
    tokenId: yesTokenId,
    makerAmount: ethers.utils.parseUnits("100", 6),  // 100 USDC
    takerAmount: ethers.utils.parseUnits("150", 6),  // 150 YES tokens
    expiration: Math.floor(Date.now() / 1000) + 86400,
    nonce: 0,
    feeRateBps: 0,
    side: 0,  // BUY
    signatureType: 0
};

const signature = await signOrder(order);
```

### 4. Submit to API

```typescript
await fetch("https://clob.polymarket.com/orders", {
    method: "POST",
    body: JSON.stringify({ order, signature })
});
```

---

## Security Model

### Non-Custodial
- Users never deposit funds to Polymarket
- Orders are signed off-chain, settled on-chain
- Users maintain full control via approvals

### Operator Limitations
- Cannot execute without valid signatures
- Cannot modify order terms
- Cannot prevent on-chain cancellations

### User Protections
- Can cancel orders directly on-chain
- Can revoke approvals anytime
- Can use nonce increment for bulk cancel

---

## References

- [Polymarket CLOB Introduction](https://docs.polymarket.com/developers/CLOB/introduction)
- [Polymarket CTF Exchange GitHub](https://github.com/Polymarket/ctf-exchange)
- [CTF Exchange Overview](https://github.com/Polymarket/ctf-exchange/blob/main/docs/Overview.md)
- [ChainSecurity Audit](https://www.chainsecurity.com/security-audit/polymarket-ctfexchange)
