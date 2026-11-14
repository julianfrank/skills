---
name: kiteconnect
description: "Guide for building with Zerodha's KiteConnect JS/TS SDK - placing orders, streaming ticks via WebSocket, fetching portfolio/market data, GTT triggers, and mutual funds. Use this skill whenever the user works with KiteConnect, Kite API, Zerodha trading, KiteTicker, or Indian stock market automation with Node.js, even if they don't mention the skill by name."
---

# KiteConnect JS/TS SDK

The official TypeScript/JavaScript client for Zerodha's Kite Connect API. Requires Node.js v18+.

```bash
npm install kiteconnect@latest
```

## Authentication Flow

Every session starts with an API key + secret. The user redirects to Kite login, gets back a `request_token`, then exchanges it for an `access_token`.

```ts
import { KiteConnect } from "kiteconnect";

const kc = new KiteConnect({ api_key: "your_api_key" });

// Step 1: redirect user to kc.getLoginURL()
// Step 2: after redirect back, extract request_token from query params
const session = await kc.generateSession(requestToken, apiSecret);
kc.setAccessToken(session.access_token);
```

Each KiteConnect instance represents one authenticated user. In server apps, create a new instance per request using the stored access_token.

Use `setSessionExpiryHook(cb)` to handle token expiry gracefully.

## Constructor

```ts
new KiteConnect({
  api_key: string,
  access_token?: string,
  root?: string,        // API endpoint root URL
  login_uri?: string,   // Custom login URL
  debug?: boolean,      // Log requests/responses
  timeout?: number      // Request timeout in ms
})
```

## Orders

### Place Order
```ts
await kc.placeOrder(variety, {
  exchange, tradingsymbol, transaction_type, quantity,
  product, order_type,
  price?,          // for LIMIT orders
  trigger_price?,  // for SL orders
  validity?,
  tag?,
  autoslice?: true // auto-split if qty exceeds freeze limit
});
```

Varieties: `"regular"`, `"amo"`, `"co"`, `"iceberg"`, `"auction"`

### Modify / Cancel / Exit
```ts
await kc.modifyOrder(variety, orderId, params);
await kc.cancelOrder(variety, orderId);
await kc.exitOrder(variety, orderId, params);
```

### Query
```ts
const orders = await kc.getOrders();
const history = await kc.getOrderHistory(orderId);
const trades = await kc.getTrades();
const orderTrades = await kc.getOrderTrades(orderId);
```

## Portfolio

```ts
const positions = await kc.getPositions();
const holdings = await kc.getHoldings();
await kc.convertPosition({ exchange, tradingsymbol, transaction_type, position_type, quantity, old_product, new_product });
const auctionInstruments = await kc.getAuctionInstruments();
```

## Market Data (REST)

```ts
// Instruments list for an exchange
const instruments = await kc.getInstruments("NSE");

// Quotes - pass "EXCHANGE:SYMBOL" strings
const quote = await kc.getQuote(["NSE:INFY", "BSE:RELIANCE"]);
const ohlc = await kc.getOHLC(["NSE:INFY"]);
const ltp = await kc.getLTP(["NSE:INFY"]);

// Historical candles
const candles = await kc.getHistoricalData(
  instrumentToken, interval,
  fromDate,   // "YYYY-MM-DD HH:MM:SS" or Date
  toDate,
  continuous?, // for futures
  oi?          // include open interest
);
```

Intervals: `"minute"`, `"3minute"`, `"5minute"`, `"10minute"`, `"15minute"`, `"30minute"`, `"60minute"`, `"day"`

## GTT (Good Till Triggered)

```ts
await kc.placeGTT({ trigger_type, tradingsymbol, exchange, trigger_values, last_price, orders });
await kc.modifyGTT(triggerId, params);
const gtts = await kc.getGTTs();
const gtt = await kc.getGTT(triggerId);
await kc.deleteGTT(triggerId);
```

## Margins

```ts
const margins = await kc.getMargins(segment?); // "equity" or "commodity"
const orderMargins = await kc.orderMargins(orders, mode?);
const basketMargins = await kc.orderBasketMargins(orders, considerPositions?, mode?);
```

## Mutual Funds

```ts
await kc.placeMFOrder({ tradingsymbol, transaction_type, amount?, quantity? });
await kc.cancelMFOrder(orderId);
const mfOrders = await kc.getMFOrders(orderId?);

await kc.placeMFSIP(params);
await kc.modifyMFSIP(sipId, params);
await kc.cancelMFSIP(sipId);
const sips = await kc.getMFSIPS(sipId?);

const mfHoldings = await kc.getMFHoldings();
const mfInstruments = await kc.getMFInstruments();
```

## User

```ts
const profile = await kc.getProfile();
const vcn = await kc.getvirtualContractNote(orders);
const valid = kc.validatePostback(postbackData, apiSecret);
```

---

## KiteTicker - WebSocket Streaming

Real-time market data via WebSocket. Supports auto-reconnection with exponential backoff.

```ts
import { KiteTicker } from "kiteconnect";

const ticker = new KiteTicker({
  api_key: "your_api_key",
  access_token: "your_access_token",
  reconnect: true,    // default: true
  max_retry: 50,      // default: 50, max: 300
  max_delay: 60       // seconds, default: 60, min: 5
});

ticker.connect();

ticker.on("connect", () => {
  ticker.subscribe([738561, 256265]); // instrument tokens
  ticker.setMode(ticker.modeFull, [738561, 256265]);
});

ticker.on("ticks", (ticks) => {
  // ticks is Tick[] with OHLC, volume, depth, etc.
});

ticker.on("disconnect", (error) => { /* handle */ });
ticker.on("error", (error) => { /* handle */ });
ticker.on("reconnect", (count, interval) => { /* log */ });
ticker.on("noreconnect", () => { /* max retries hit */ });
ticker.on("order_update", (order) => { /* order status change */ });
ticker.on("close", (reason) => { /* cleanup */ });
```

### Modes
- `ticker.modeFull` - full tick data (OHLC, volume, depth, OI, timestamps)
- `ticker.modeQuote` - quote-level data
- `ticker.modeLTP` - last traded price only

### Methods
```ts
ticker.connect();
ticker.disconnect();
ticker.connected();                        // returns boolean
ticker.subscribe(tokens);                  // number[] or string[]
ticker.unsubscribe(tokens);
ticker.setMode(mode, tokens);
ticker.autoReconnect(enable, maxRetry?, maxDelay?);
```

## Common Patterns

### Graceful Reconnection
```ts
ticker.autoReconnect(true, 20, 5);
ticker.on("reconnect", (count, interval) => {
  console.log(`Reconnecting: attempt ${count}, next in ${interval}s`);
});
ticker.on("noreconnect", () => {
  console.log("Max reconnection attempts reached");
  process.exit(1);
});
```

### Instrument Format
REST endpoints use `"EXCHANGE:SYMBOL"` (e.g. `"NSE:INFY"`).
WebSocket uses numeric instrument tokens (e.g. `256265`).
Get instrument tokens from `kc.getInstruments(exchange)`.
