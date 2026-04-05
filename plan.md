# Best Kaspit — GitHub Pages Deployment Plan

## Goal
Convert the existing desktop Kaspit fund comparison tool into a static website deployable on GitHub Pages. Data is fetched client-side (in-browser) — no server, no Selenium, no file downloads.

## Key Discovery
The source page (`https://www.funder.co.il/kaspit`) embeds all fund data as a JavaScript variable:
```js
var kaspitData = {"x":[ { fundNum, fundName, 1day, monthBegin, yearBegin, nihol, hosafa, lastUpdate, y30, ... }, ... ]};
```
This JSON is available in the raw HTML — no need for Selenium or headless browser. A simple `fetch()` + regex/parse can extract it.

## Architecture

```
best kaspit/
├── index.html          # Single-page app (HTML + embedded CSS + JS)
├── .github/
│   └── workflows/
│       └── deploy.yml  # GitHub Pages deploy workflow
└── plan.md             # This file
```

Single `index.html` file — no build step, no dependencies, no framework.

---

## Steps

### Step 1 — Data Fetching (in-browser, in-memory)
- Use a CORS proxy (e.g. `https://corsproxy.io/?` or `https://api.allorigins.win/raw?url=`) to fetch the funder.co.il/kaspit HTML from the browser (the site doesn't set CORS headers).
- Extract the `kaspitData` JSON from the HTML using a regex: `var kaspitData\s*=\s*({.*?});`
- Parse the JSON to get the array of fund objects.
- Extract `lastUpdate` from the first fund entry (all share the same date).
- **Fallback**: If the CORS proxy fails, show a clear error message.

### Step 2 — Data Processing (mirrors existing Python logic)
Map each fund object to the display model:
```
Fields from kaspitData.x[]:
  fundName     → name
  1day         → daily
  monthBegin   → from_start_of_month
  yearBegin    → from_start_of_year (YTD)
  y30          → monthly_return (last 30 days)
  nihol        → management_fee
  hosafa       → addition_rate
  (computed)   → net_monthly = y30 - nihol - hosafa
```

Sort by `net_monthly` descending, tiebreak by `yearBegin` descending.
Filter out funds with null/missing `monthBegin` or `yearBegin` (expired/not-yet-active funds).

### Step 3 — UI (HTML table)
- RTL Hebrew layout (`dir="rtl"`)
- Responsive table with alternating row colors
- Column headers matching the existing UI:
  | שם קרן | יומי (%) | מתחילת החודש (%) | מתחילת השנה (%) | תשואה חודשית (%) | דמי ניהול (%) | שיעור הוספה (%) | נטו חודשי (%) |
- Color-code `net_monthly`: green for positive, red for negative
- **"Last Updated: DD/MM/YYYY"** label displayed prominently above the table, extracted from the data's `lastUpdate` field
- Loading spinner while data is being fetched
- Click-to-copy on any cell (same UX as the tkinter version)
- Mobile-friendly (horizontal scroll on small screens)

### Step 4 — GitHub Pages Deployment
- Initialize git repo in `D:\workspace\best kaspit`
- Create `.github/workflows/deploy.yml` using the `actions/deploy-pages` workflow
- Push to GitHub → auto-deploys `index.html` from root

### Step 5 — Polish
- Add page title and favicon
- Error state UI (if fetch fails)
- "Source: funder.co.il/kaspit" attribution link at bottom
