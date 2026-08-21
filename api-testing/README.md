# API Testing

Investment Tracker talks to Firebase Firestore through the Firebase SDK (not a REST API the app exposes), and to a third-party RapidAPI service ("Yahoo Finance" on RapidAPI) for prices, historical chart data, and news. This folder tests that third-party dependency, since it's the one testable surface.

## How to use the collection
1. Import `postman-collection.json` into Postman.
2. Set the `rapidapi_key` collection variable to a valid RapidAPI key for the `yahoo-finance166` API (the placeholder value will fail with a 403).
3. Run the collection (or individual requests) — each request has built-in `pm.test` assertions.

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

## Observation
The "missing required parameter returns 200 instead of 400" behavior is worth keeping an eye on: since the app's own client-side validation (`validateStockSymbol` in `watchlist.js`) relies on this API returning a usable price for a symbol to decide if it's valid, any looseness in the API's own validation could let unexpected input through the app's "invalid symbol" check in edge cases. Not filed as a bug against the app itself — it's third-party API behavior — but flagged here as a risk to watch.
