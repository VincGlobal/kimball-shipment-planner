# Kimball Shipment Planner — Context Notes

## What it is
A standalone HTML tool (`shipment-planner.html`) used by Global O-Ring to plan outbound shipments for the Kimball customer. No server, no build step — just open the file in a browser or visit the hosted URL.

## Where it lives
- **Local file:** `C:\vinc_code\kimball_shipment_planner\shipment-planner.html`
- **GitHub repo:** https://github.com/VincGlobal/kimball-shipment-planner
- **Live URL:** https://vincglobal.github.io/kimball-shipment-planner/shipment-planner.html

## How to deploy changes
Edit `shipment-planner.html` locally, then push to the `master` branch. GitHub Pages updates automatically within a minute or two.

For the proxy (`kimball-proxy`), deploy via: `cd C:\vinc_code\kimball-proxy && npx vercel --prod`

## What the tool does

### Open Order Picker (top bar)
- On page load, fetches all open KIMMID orders live from the ERP via `/api/open-orders` on the proxy
- Dropdown shows each order as: `829678  —  IM300868  —  4800 ROBERTS ROAD  (150 open / 288 lines)`
- Selecting an order instantly fills **Sales Order #**, **Customer PO #**, and **Ship To**, then automatically fetches and loads that order's line items via `/api/order-lines` on the proxy (see below) — no manual CSV export/drop needed for the common path
- Ship To matching normalizes address abbreviations (ROAD→RD, BOULEVARD→BLVD, etc.) to match the hardcoded dropdown options
- Falls back gracefully with a "Could not load orders" message if the proxy is unreachable
- Proxy endpoint: `GET /api/open-orders` — queries ERP for `customer.customer=KIMMID&open=1`, returns `[{ orderNumber, po, address, openLines, totalLines }]`

### Order Lines Auto-Load (replaces manual CSV export for the common path)
- Proxy endpoint: `GET /api/order-lines?orderNumber={n}` — paginates through `/api/order_lines?order.orderNumber={n}` in the ERP (500/page) and returns `{ rows: [{ line, item, com, price, total, open }] }`
- The tool converts those rows into the same minimal CSV shape (`Line,Item,Com,Price,Total,Open`) the manual export produces and feeds it straight into the existing `processCSV()` pipeline — same shipment-building/rendering code path regardless of source
- Only these 6 columns are ever actually read by the tool's logic, even from a manually-exported CSV with all ~24 columns — everything else in that export (Notes, Image, UM, SC, WO, BO, PO, TO, FO, MP, Cost, MC, GP%, Tax, Wanted Date, Item Block) is cosmetic/unused
- The order picker dropdown disables itself while the fetch is in flight and re-enables after

### CSV Shipment Planner (main panel)
- The manual CSV drop zone is now a small fallback control next to the Open Order picker (not the full-page drop zone it used to be) — drop or browse a sales order CSV exported from 10X ERP (the download icon in the Line Items toolbar on the Sales Order detail page, `app.10xerp.com/sales-orders/{orderNumber}`)
- Full CSV columns in that manual export: `Line, Notes, Image, Item, Description, UM, Ord, Com, Ship, SC, WO, BO, PO, TO, FO, Price, MP, Cost, MC, GP%, Total, Tax, Wanted Date, Open, Item Block` — but only `Item`, `Com` (committed qty), `Price`, `Total` (pre-calculated line $), `Open` (Yes/No), and `Line` are actually used (see above)
- Splits committed lines (Com > 0) into shipments of max 25 lines each
- **Simple mode** (< 50 committed lines): all lines go into standard shipments
- **Full mode** (≥ 50 lines): separates autobagger-eligible items from standard items
- Autobagger eligibility: item must match `KM-[MATERIAL][SIZE]/[QTY]`, material in `{N, C, V, BV, HSN, HNBR, QN}`, size in the standard AS568 size list, and Com > 30
- KIT and GAGE items are tracked separately (not counted in bag totals)
- Each shipment card shows Lines, Bags, Revenue and can be individually copied as a TSV row ready to paste into the ERP
- "Copy All Shipments" copies all rows at once
- Recently viewed orders are saved to localStorage (last 10) — regardless of whether they were loaded via the order picker or a manual CSV drop

### Summary bar (top dashboard)
- **Total Lines** — all lines in the CSV (open + closed), with `X OPEN · Y CLOSED` subtitle in cyan
  - Open = `Open` column = "Yes" (still needs to ship or awaiting inventory)
  - Closed = `Open` column = "No" (already shipped/fulfilled)
- **Total Revenue** — sum of the `Total` column for ALL lines (open and closed), representing the full SO value once completely filled
- **No Commit (Skipped)** — lines where Com = 0; these are open lines waiting on inventory availability
- Full mode also shows Auto/Standard breakdowns (committed lines only)

### Set Ship Date sidebar (middle panel)
- Persistent sidebar panel between the main content and the Location Parser
- Lists all shipments by number with checkboxes — select which ones to update
- Shipment numbers auto-update in the list when edited in any card (no manual refresh needed)
- Pick a date and click "Update Shipments" to PATCH `wantedDate` in the ERP for all selected shipments sequentially
- Shows elapsed time ticker during updates and a per-shipment summary on completion
- **Avg Update Time/Shipment** — lifetime average displayed at the bottom of the panel, persisted to Upstash Redis via `/api/stats` on the proxy; shared across all computers and browsers
- Panel pulses with neon green glow while updates are in progress

### Location Parser (right sidebar)
- Drop a Picking Slip PDF (uses PDF.js 3.11.174 via CDN)
- Extracts warehouse locations per line item and classifies them:
  - `01/02/03` → VLM
  - `0` → RECEIVING
  - `KITS-*` → KITS
  - `S-M` or `S-[n]` → PALLETS
  - Everything else → OVERFLOW
  - Description contains CORD or SPLICING KIT → SHOP
- Handles both "Item Components" and "Lot Details" PDF formats (Kimball uses Lot Details)
- Flags conflicts when a single line has locations in multiple categories — user resolves each one
- Generates a formatted warehouse/kit room note with bold and italic styling
- "Copy Note" copies rich text (bold/italic preserved) for pasting directly into the ERP
- "Autobag?" checkbox adds an auto-bag instruction to the note

### Toolbar buttons (top of Location Parser sidebar)
- **Copy Date:** copies today's date + 7 days (MM/DD/YYYY) to clipboard
- **Locations:** copies the location summary string (e.g. "19 VLM / 3 OVERFLOW / 1 KIT ROOM")

### Other UI
- Animated wavy grid background (canvas)
- **Pause button** (bottom-left, fixed): pauses/resumes the background animation
- SO#, Customer PO#, and Ship To fields feed into the copied TSV rows
- Ship To dropdown has 5 Kimball addresses hardcoded

## Kimball ship-to addresses (hardcoded in dropdown)
- 4800 ROBERTS RD
- 1501 E BARDIN RD
- 255 S MCCARRAN BLVD
- 14 PROSPECT DR
- 730 KING GEORGE BLVD

## ERP Date Integration

Each shipment card has a **Set Date** button. Clicking it opens a modal to pick a date and update one shipment. The **Set Ship Date** sidebar panel lets you update multiple shipments at once. Both PATCH `wantedDate` on all lines in the selected shipment(s) in 10X ERP.

### How it works
- The HTML tool cannot call the ERP directly (CORS — the ERP returns no CORS headers)
- A Vercel serverless proxy sits in between: `https://kimball-proxy.vercel.app/api/update-dates`
- The proxy holds the ERP API key as an encrypted Vercel environment variable (`ERP_API_KEY`)
- The tool POSTs `{ orderNumber, lineNumbers, wantedDate }` to the proxy
- The proxy fetches all order lines from the ERP, finds the ones matching the line numbers, and PATCHes each one

### ERP details
- **ERP:** 10X ERP (PHP/Symfony, API Platform) at `https://app.10xerp.com` (production — migrated from the `poc4.10xerp.com` dev/sandbox environment on 2026-07-02; same API shape, just a different tenant + API key)
- **Auth:** `X-Auth-Token: <key>` header
- **Endpoint used:** `GET /api/order_lines?order.orderNumber={n}` to find lines, then `PATCH /api/order_lines/{id}` with `Content-Type: application/merge-patch+json` and body `{ "wantedDate": "YYYY-MM-DD" }`
- The `line` column in the CSV maps directly to `order_line.line` in the ERP

### Proxy repo
- **Local:** `C:\vinc_code\kimball-proxy`
- **Deployed to:** Vercel personal account (vincglobal), NOT the company Vercel account
- **API key:** stored in Vercel env as `ERP_API_KEY` — not visible in code or logs
- **Stats endpoint:** `/api/stats` — GET returns `{ total_ms, total_ships }`; POST increments them. Backed by Upstash Redis (KV store connected to the kimball-proxy Vercel project, named `upstash-kv-beige-paddle`).

## Other UI details
- CSV file loads immediately on drop, but is hidden behind an overlay until SO#, Customer PO#, and Ship To are all filled in — once they are, the overlay clears and the shipments render with those values applied
- "Load New File" clears all three top fields and resets the tool
- Shipment sequence numbers are editable — changing the first shipment's number cascades to all following ones; the Set Ship Date panel auto-updates to reflect any edits
- TSV copy rows do NOT include the shipment number suffix (just `{SO#}-\t...`)

## Tech stack
- Vanilla HTML/CSS/JS — no framework, no build tools
- PDF.js 3.11.174 (CDN) for picking slip parsing
- Google Fonts: Lexend (loaded via CDN)
- localStorage for recently viewed orders
- Upstash Redis (via Vercel KV) for cross-device avg update time stat
