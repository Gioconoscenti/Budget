# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the app

No build step. Open `bilancio-personale.html` directly in a browser.

All external dependencies are loaded from CDN at runtime:
- Chart.js 4.4.1 (`cdnjs.cloudflare.com`)
- Google Fonts: Syne + Space Mono

## Architecture

The entire app is a **single HTML file** (~3600 lines) with inline CSS and JavaScript. No framework, no bundler, no package manager.

### Data layer — Google Sheets backend

All persistence goes through a Google Apps Script Web App deployed as a public endpoint. The URL is stored in `localStorage` (`bp_gs_url`) and in the `GS_URL` variable.

Every operation uses **GET requests** (never POST) to avoid CORS preflight issues:
```
GET {GS_URL}?action=<action>&payload=<json-encoded>
```

Key actions: `read`, `write`, `replace`, `sessions`, `config`, `log`, `getRules`, `getBudRules`, `getInv`, `addInv`, `deleteInv`, `updateRow`, `saveRules`, `saveBudRules`, `saveConfig`, `addLog`.

Large writes are chunked in batches of 100 rows: the first chunk uses `replace` (clears the sheet), subsequent chunks use `write` (append).

### Data model

A transaction object has:
- `data` (Date), `dataStr` (DD/MM/YYYY string)
- `descrizione`, `tipologia`, `importo` (signed float), `tipo` (`entrata`/`uscita`)
- `conto` (`ordinario`/`straordinario`)
- `acquirente` (`gioacchino`/`alessandra`)
- `budget` (`Necessità`/`Tempo libero`/empty)
- `source` (`csv`/`mediolanum`/`postepay`/`json`)

### State

Global variables hold all runtime state:
- `allTx` — merged, deduplicated transaction array (source of truth for charts/tables)
- `csvTx` / `apiTx` — transactions from Google Sheets / from bank API respectively
- `pendingTx` — new transactions awaiting approval, persisted in `localStorage('bp_pending')`
- `allInv` — investments array, loaded from GS on section entry

### Transaction categorization

`autoCateg(desc, importo)` applies a three-tier lookup:
1. Exact match in user rules (`userExpRules` / `userIncRules`) stored in `localStorage` and synced to GS
2. Exact match in `CSV_RULES` — a hardcoded dictionary of ~548 keyword→category mappings
3. Partial substring match (longest keyword first) against the same rule sets

`getBudget(tipologia)` maps a category to `Necessità` or `Tempo libero` via `userBudRules` (user-overridable) falling back to `BUDGET_MAP` (hardcoded).

User rules take precedence over `CSV_RULES`. They are saved to `localStorage` immediately and synced to GS asynchronously.

### Sections

Navigation calls `go(id, btn)`. Each section has a `<div class="section" id="sec-<id>">` and is activated by adding `.active`. Sections:

| id | Description |
|---|---|
| `overview` | KPI cards + charts + AI panel |
| `flussi` | Monthly/quarterly/yearly cash flow charts |
| `categorie` | Category breakdowns + 50/30 budget analysis |
| `transazioni` | Paginated transaction table with inline editing |
| `conto` | P&L statement (quarterly/annual, 3-level expandable) |
| `investimenti` | Investment portfolio loaded from GS |
| `banche` | Open Banking connections via Cloudflare Worker + Enable Banking |
| `importa` | CSV import (Mediolanum/Postepay) + Postepay screenshot OCR |
| `regole` | Keyword→category and category→budget rule management |

### AI panel

`askAI()` sends a natural-language question about the transaction data to the **Gemini API** (key hardcoded as `GEMINI_KEY`). `buildFinancialContext()` assembles a text summary of all transactions (monthly aggregates, top categories, top transactions) and includes it in the prompt. The app auto-discovers the best available Gemini model by probing candidates in order (`gemini-2.0-flash` first).

### Open Banking

Bank connections use a **Cloudflare Worker** as a CORS proxy (`workerUrl`), which authenticates with Enable Banking using an RSA private key stored as a Worker secret. Sessions (OAuth tokens) and account lists are persisted on the GS backend.

### Filters

Period filter (`period` variable), account filter (`contoG`), acquirente filter (`acquirenteG`), and source filter (`sourceG`) are global and applied by `filterTx()` before every render. `applyFilters()` also handles the transaction-table-specific filters (type, category, date range, search).

### Date parsing

`parseD(s)` handles: ISO strings, Italian format (DD/MM/YYYY), Excel serial numbers, and JS Date strings. Always converts to local midnight to avoid timezone shifts.
