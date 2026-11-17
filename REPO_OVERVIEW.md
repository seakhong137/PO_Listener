# pos-payway — Repository Overview

**Repo name:** `pos-payway` (v1.0.0) · **Runtime:** Node.js (ES modules) + static single-file HTML apps
**Backing cloud:** Firebase project `posystem-3e373` (Firestore + Auth) · **Hosting target:** Render (`https://pos-payway.onrender.com`)

This repository powers a small POS ecosystem for Cambodian shops that accept **ABA PayWay** payments — both *indoor* (ABA Bank app, "ABA PAY") and *outdoor* (external banks scanning **KHQR** codes). It has two halves:

1. **A backend bridge (`listener.mjs`)** that listens to the shop's Telegram groups via a userbot, parses ABA payment confirmation messages in real time, and exposes them as an HTTP/SSE payment feed.
2. **Three front-end web apps** (single large HTML files with React/Tailwind loaded from CDNs): the main POS/Purchase Order system, a BI dashboard, and a customer-facing KHQR display screen.

---

## 1. `listener.mjs` — ABA PayWay Telegram Group Listener Bridge (~900 lines)

The core service. It connects to Telegram's MTProto API using the **GramJS** client with a pre-authenticated user session, watches whitelisted shop chat groups, and extracts structured payment data from ABA bank notification messages.

### How it works

| Stage | Detail |
|---|---|
| **Auth** | Requires env vars `TG_API_ID`, `TG_API_HASH`, `TG_SESSION` (GramJS StringSession). Exits if missing. A helper page for generating `TG_SESSION` is embedded in `index.html`. |
| **Chat whitelist** | Chat IDs come from `PAYWAY_CHAT_ID` env var **and** dynamically from Firestore collections `shops` and `payway_secrets` (refreshed every 10 min or instantly via `/sync-config`). IDs are normalized (stripping `-100` prefixes). |
| **Parsing** | `parseABAPayWay()` matches ABA notification text (strict regex + fallback regex), extracting: currency (KHR/USD), amount, payer name, masked account, paid timestamp, payment channel, source bank, merchant name, **Trx ID**, and **APV code**. Messages tagged `INDOOR_ABA` vs `OUTDOOR_KHQR`. Timestamps are aligned to minute granularity against message server time. |
| **Storage — "Smart Firebase Bypass"** | By default (`BYPASS_FIREBASE_DB=true`) transactions are **never written to Firestore**; they live only in a 2,000-item in-memory ring buffer served over SSE/HTTP (0 Firebase writes). When bypass is OFF (globally or per-chat via `bypassFirebaseDb === false` on a shop doc), payments are written to Firestore collection `aba_payments`. A circuit breaker pauses writes for 10 minutes on `RESOURCE_EXHAUSTED` quota errors. |
| **Live capture** | Two handlers: a `NewMessage` event handler plus a raw MTProto update fallback; plus a periodic sweep (`every 3.5 s`) of recent messages and a 120-message backfill at boot. |
| **Deep history** | `fetchHistoricalGroupMessages()` paginates backwards through Telegram group history for arbitrary time ranges (up to ~2,000 msgs/chat) to backfill missed payments. |

### HTTP API (port `$PORT` or `10000`; CORS wide open)

| Endpoint | Purpose |
|---|---|
| `GET /events` (`/stream`, `/sse`) | Server-Sent Events: pushes each new parsed payment live; initial burst includes last 120 payments; 20 s heartbeat comments. |
| `GET /recent` / `/payments` / `/` | JSON feed of buffered payments (filters: `chatId`, `startMs`, `endMs`, `limit`; `?scan=1` triggers a fresh sweep). Also reports status, allowed chats, bypass map, quota state. |
| `GET /history?chatId&startMs&endMs&limit` | Deep historical Telegram backfill → array of parsed payments. |
| `GET /claim?trxId&orderId&invoiceNo` | Marks a payment as matched to an order/invoice (in memory only). |
| `GET /delete?trxIds` | Removes entries from the in-memory buffer. |
| `GET /sync-config` | Instantly reloads the shop whitelist & per-chat bypass flags from Firestore. |

A self keep-alive ping hits `RENDER_EXTERNAL_URL` every 4 minutes so the free Render instance doesn't sleep.

---

## 2. `index.html` — "Purchase Order System" — Main POS Terminal (~2 MB, single file)

The flagship app: a PWA-installable, React-based POS & back-office system for shop staff (cashiers/managers). Key characteristics discovered in the code:

- **Stack:** React 18 + Babel standalone + Tailwind CDN, Firebase v9 compat SDK (Auth + Firestore), Khmer font support (Khmer/English bilingual UI).
- **Data model (Firestore):** `products`, `purchaseOrders`, `vendors`, `customers`, `users`, `shifts`, `sales`, `productMovement`, `internalTransfers`, `crossPromoProducts`, `costHistory`, plus `aba_payments`.
- **Feature areas:** vendor & purchase-order management, stock/inventory tracking and transfers, cashier sales, shift management, reporting/dashboards, printable receipts (embedded "Print Receipt" document), and a built-in **Telegram `TG_SESSION` generator** tool page.
- **Payment reconciliation:** heavily integrates ABA PayWay/KHQR (hundreds of references). It consumes the `listener.mjs` endpoints (`/recent`, `/history`, `/events` at `pos-payway.onrender.com`) for live payment detection, falling back to reading Firestore `aba_payments` directly when payments are persisted there.

## 3. `POS-Dash.html` — POS Business Intelligence Dashboard

Read-only analytics/reporting front-end (React + Tailwind + Font Awesome, Firebase compat SDK, SheetJS/XLSX export). Reads `aba_payments` (and related collections) to produce management reports and exports (title switches per report module: "POS BI Management Report").

## 4. `POS-KHRQ.html` — Customer Display Kiosk ("Live Cart & ABA KHQR")

Second-screen customer-facing display for checkout counters: shows the live cart and renders the **ABA KHQR** payment QR code, then confirms payment arrival. Uses Firebase Auth/Firestore for session/cart state and KHQR generation logic.

---

## Configuration Reference (`listener.mjs`)

| Env var | Default | Meaning |
|---|---|---|
| `TG_API_ID` / `TG_API_HASH` / `TG_SESSION` | — (required) | Telegram MTProto credentials & userbot session |
| `PAYWAY_CHAT_ID` | empty | Comma/space-separated whitelisted group chat IDs |
| `BYPASS_FIREBASE_DB` | `true` | ON = memory+SSE only (0 writes); OFF = persist to `aba_payments` |
| `FIREBASE_SERVICE_ACCOUNT(_JSON)` | — | Service-account JSON; falls back to Application Default Credentials |
| `PORT` | `10000` | HTTP/SSE server port |
| `RENDER_EXTERNAL_URL` | `https://pos-payway.onrender.com` | Keep-alive ping target |

Per-shop overrides live in Firestore docs (`shops.bypassFirebaseDb`, `payway_secrets`).

## Running

```bash
npm install
npm start          # = node listener.mjs  (needs the env vars above)
```

The three `.html` files are self-contained front-ends deployable to any static host (they call the listener service and Firebase directly).

## Notable Design Decisions & Caveats

- **Firestore-free hot path by default**: the "Smart Bypass" exists to stay inside Firebase's free-tier write quota while still handling 1,000+ daily transactions via the in-memory ring buffer. Consequence: **payments are lost on process restart** unless re-backfilled via `/history` (which the UI can trigger) or unless bypass is disabled for that chat.
- **Claim/delete state is memory-only**, so order-matching resets on redeploy.
- The HTTP API has **no authentication** and `Access-Control-Allow-Origin: *` — anyone who knows the Render URL can read, claim, or delete payments. Fine for a private URL, risky otherwise.
- Front-ends embed Firebase web config and run Babel in-browser (large single files, heavy cold start) — typical of fast-iterated prototype architecture rather than a build pipeline.
- GramJS userbot approach means the service impersonates a real Telegram account that must be a member of the shop notification groups.
