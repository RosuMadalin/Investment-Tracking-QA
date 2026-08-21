# API Testing

Investment Tracker talks to Firebase Firestore through the Firebase SDK (not a REST API the app exposes), and to a third-party RapidAPI service ("Yahoo Finance" on RapidAPI) for prices, historical chart data, and news. This folder tests that third-party dependency, since it's the one testable surface.

## How to use the collection

**In the Postman app:**
1. Import `postman-collection.json` into Postman.
2. Set the `rapidapi_key` collection variable's *Current Value* (not *Initial Value*, so it never gets re-exported) to a valid RapidAPI key for the `yahoo-finance166` API — the placeholder value will fail with a 401.
3. Run the collection (or individual requests) — each request has built-in `pm.test` assertions.

**From the command line, with Newman** (Postman's official CLI runner — this is what runs a collection in a CI/CD pipeline instead of a person clicking "Run" in the app):
```
npm install -g newman
newman run postman-collection.json --env-var "rapidapi_key=<your-key>"
```
Or with the environment file: `newman run postman-collection.json -e postman-environment.json` (after filling in the key there instead).

**Note on the key:** the real key is intentionally *not* committed here — it's a live credential and the file is meant to be public on GitHub. As a side note unrelated to this test scope: the app's own frontend code currently hardcodes this key in plain JS (visible in the deployed site's source), which is a separate finding worth addressing later even though security testing is out of scope for this project.

## Endpoints covered

| Endpoint | Purpose |
|----------|---------|
| `GET /api/stock/get-price` | Current price for a symbol (used by the watchlist) |
| `GET /api/stock/get-chart` | Historical candlestick data for a symbol/range (used by the charts) |
| `GET /api/news/list-by-symbol` | News articles for one or more symbols (used by the Home/news view) |

## Results (executed 2026-08-19, against the live API, real responses)

| Case | Status | Result |
|------|--------|--------|
| Valid symbol price (AAPL) | 200 | Returns full quote object with `regularMarketPrice` |
| Invalid symbol price (ZZZZZ) | 404 | Clean error body: `{"error":true,"message":{...,"error":{"code":"Not Found", ...}}}` |
| Missing `symbol` param | 200 (unexpected) | API does **not** reject the request — it returned a 200 with quote data instead of a 400. Documented as a known quirk of the third-party API, not something the app can control. |
| Valid chart (AAPL, 1mo) | 200 | Returns `chart.result[0]` with timestamps + OHLC arrays |
| Valid news (AAPL) | 200 | Returns `data.main.stream` array of articles |

## Newman CLI run (executed 2026-08-21)

```
$ newman run postman-collection.json --env-var "rapidapi_key=***"

Investment Tracker - API Testing

□ Price
└ GET price - valid symbol (AAPL) [200 OK]
  √ Status code is 200
  √ Response has a regularMarketPrice
└ GET price - invalid symbol (ZZZZZ) [404 Not Found]
  √ Status code is 404 for unknown symbol
  √ Error body reports Not Found
└ GET price - missing symbol param [200 OK]
  √ Response status is documented (200 observed, not 400)

□ Historical Chart
└ GET chart - valid symbol + range (AAPL, 1mo) [200 OK]
  √ Status code is 200
  √ Response contains chart result with timestamps
└ GET chart - invalid symbol [404 Not Found]
  √ Request completes (status documented, not assumed)

□ News
└ GET news - valid symbol (AAPL) [200 OK]
  √ Status code is 200
  √ Response contains a news stream array
└ GET news - missing symbol param [200 OK]
  √ Request completes (status documented, not assumed)

  assertions   11 executed, 0 failed
  total run duration: 4.8s
```

11/11 assertions passed. Note: running the same suite repeatedly in a short window (as happened once during development, with 9 rapid-fire calls) triggered `429 Too Many Requests` from the third-party API — a reminder that this is a rate-limited free-tier API, which matters if this collection is ever wired into a CI pipeline that runs on every commit.

## Observation
The "missing required parameter returns 200 instead of 400" behavior is worth keeping an eye on: since the app's own client-side validation (`validateStockSymbol` in `watchlist.js`) relies on this API returning a usable price for a symbol to decide if it's valid, any looseness in the API's own validation could let unexpected input through the app's "invalid symbol" check in edge cases. Not filed as a bug against the app itself — it's third-party API behavior — but flagged here as a risk to watch.
