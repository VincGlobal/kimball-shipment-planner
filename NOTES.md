# Kimball Shipment Planner — Context Notes

## What it is
A standalone HTML tool (`shipment-planner.html`) used by Global O-Ring to plan outbound shipments for the Kimball customer. No server, no build step — just open the file in a browser or visit the hosted URL.

## Where it lives
- **Local file:** `C:\vinc_code\kimball_shipment_planner\shipment-planner.html`
- **GitHub repo:** https://github.com/VincGlobal/kimball-shipment-planner
- **Live URL:** https://vincglobal.github.io/kimball-shipment-planner/shipment-planner.html

## How to deploy changes
Edit `shipment-planner.html` locally, then push to the `master` branch. GitHub Pages updates automatically within a minute or two.

## What the tool does

### CSV Shipment Planner (main panel)
- Drop or browse a sales order CSV (columns: Line, Item, Com, Price)
- Splits committed lines into shipments of max 25 lines each
- **Simple mode** (< 50 committed lines): all lines go into standard shipments
- **Full mode** (≥ 50 lines): separates autobagger-eligible items from standard items
- Autobagger eligibility: item must match `KM-[MATERIAL][SIZE]/[QTY]`, material in `{N, C, V, BV, HSN, HNBR, QN}`, size in the standard AS568 size list, and Com > 30
- KIT and GAGE items are tracked separately (not counted in bag totals)
- Each shipment card shows Lines, Bags, Revenue and can be individually copied as a TSV row ready to paste into the ERP
- "Copy All Shipments" copies all rows at once
- Recently viewed orders are saved to localStorage (last 10)

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

### Toolbar buttons (bottom of sidebar)
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

Each shipment card has a **Set Date** button. Clicking it opens a modal where you pick a date, then click "Update ERP." This PATCHes the `wantedDate` field on every line in that shipment directly in the 10X ERP — controlling which lines print (and therefore ship).

### How it works
- The HTML tool cannot call the ERP directly (CORS — the ERP returns no CORS headers)
- A Vercel serverless proxy sits in between: `https://kimball-proxy.vercel.app/api/update-dates`
- The proxy holds the ERP API key as an encrypted Vercel environment variable (`ERP_API_KEY`)
- The tool POSTs `{ orderNumber, lineNumbers, wantedDate }` to the proxy
- The proxy fetches all order lines from the ERP, finds the ones matching the line numbers, and PATCHes each one

### ERP details
- **ERP:** 10X ERP (PHP/Symfony, API Platform) at `https://poc4.10xerp.com`
- **Auth:** `X-Auth-Token: <key>` header
- **Endpoint used:** `GET /api/order_lines?order.orderNumber={n}` to find lines, then `PATCH /api/order_lines/{id}` with `Content-Type: application/merge-patch+json` and body `{ "wantedDate": "YYYY-MM-DD" }`
- The `line` column in the CSV maps directly to `order_line.line` in the ERP

### Proxy repo
- **Local:** `C:\vinc_code\kimball-proxy`
- **Deployed to:** Vercel personal account (vincglobal), NOT the company Vercel account
- **API key:** stored in Vercel env as `ERP_API_KEY` — not visible in code or logs

## Other UI details
- CSV file loads immediately on drop, but is hidden behind an overlay until SO#, Customer PO#, and Ship To are all filled in — once they are, the overlay clears and the shipments render with those values applied
- "Load New File" clears all three top fields and resets the tool
- Shipment sequence numbers are editable — changing the first shipment's number cascades to all following ones
- TSV copy rows do NOT include the shipment number suffix (just `{SO#}-\t...`)

## Tech stack
- Vanilla HTML/CSS/JS — no framework, no build tools
- PDF.js 3.11.174 (CDN) for picking slip parsing
- Google Fonts: Lexend (loaded via CDN)
- localStorage for recently viewed orders
