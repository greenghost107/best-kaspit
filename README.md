# קרנות כספיות — השוואה

A static, single-page web app that compares Israeli money market funds (קרנות כספיות) ranked by net monthly return after all fees. Deployed on GitHub Pages — no server, no backend, no build step.

Live site: `https://<your-username>.github.io/<repo-name>/`

---

## What it does

Fetches live data from [funder.co.il/kaspit](https://www.funder.co.il/kaspit), extracts the embedded fund JSON entirely in the browser, and renders a ranked comparison table. No files are downloaded or cached to disk.

### Ranking formula

```
net_monthly = monthly_return − management_fee − addition_rate
```

Funds are sorted descending by `net_monthly`, with YTD as a tiebreaker. This surfaces the fund that puts the most money in your pocket each month after all costs.

### Columns

| Column | Hebrew | Source field |
|--------|--------|--------------|
| # | מיקום | computed rank |
| Fund Name | שם קרן | `fundName` |
| Daily % | יומי | `1day` |
| Month-to-Date % | מתחילת החודש | `monthBegin` |
| YTD % | מתחילת השנה | `yearBegin` |
| Monthly Return % | תשואה חודשית | `y30` (last 30 days) |
| Mgmt Fee % | דמי ניהול | `nihol` |
| Addition Rate % | שיעור הוספה | `hosafa` |
| **Net Monthly %** | **נטו חודשי** | `y30 − nihol − hosafa` |

Funds with missing `monthBegin` or `yearBegin` (expired or not-yet-active funds) are excluded.

---

## How the data is fetched

GitHub Pages serves only static files — there is no server to make backend requests. The page works around this using a **CORS proxy**:

```
Browser → CORS proxy → funder.co.il/kaspit → HTML response → back to browser
```

Two proxies are tried in order, with a 20-second timeout each:

1. `https://corsproxy.io/` (primary)
2. `https://api.allorigins.win/` (fallback)

If both fail, an error message is shown with a retry button.

### Data extraction (in-memory)

The source page embeds all fund data as a JavaScript variable:

```js
var kaspitData = {"x": [ { fundNum, fundName, y30, nihol, hosafa, ... }, ... ]};
```

The app locates this variable in the raw HTML by string index and parses the JSON slice directly — no DOM parsing, no file I/O, no caching.

---

## UI features

- **Last Updated** badge in the header — derived from the most recent `lastUpdate` field in the fund data
- **Net Monthly** column is colour-coded: green (positive), red (negative)
- **Top 3 rows** have gold/silver/bronze accent borders on the Net Monthly cell
- **Click any cell** to copy its value to the clipboard — a blue flash and toast notification confirm the copy
- **Refresh button** re-fetches live data on demand
- Horizontally scrollable table on small screens (mobile-friendly)
- Full RTL Hebrew layout

---

## Project structure

```
best kaspit/
├── index.html                  # Entire app — HTML, CSS, and JS in one file
├── .github/
│   └── workflows/
│       └── deploy.yml          # GitHub Actions: auto-deploy to GitHub Pages on push to main
├── .nojekyll                   # Disables GitHub Pages Jekyll processing
├── .gitignore
├── plan.md                     # Original implementation plan
└── README.md
```

No `package.json`, no `node_modules`, no build output — `index.html` is the artifact.

---

## Running locally

No install or build required. Open `index.html` directly in a browser:

```bash
# Option 1 — direct file open (may block clipboard API in some browsers)
open index.html

# Option 2 — simple local server (recommended, enables full clipboard support)
python -m http.server 8080
# then visit http://localhost:8080
```

Or with Node.js:

```bash
npx serve .
```

---

## Deploying to GitHub Pages

### First-time setup

```bash
cd "path/to/best kaspit"
git init
git add .
git commit -m "Initial deploy"
git remote add origin https://github.com/<your-username>/<repo-name>.git
git push -u origin main
```

Then in the GitHub repo:

1. Go to **Settings → Pages**
2. Under **Source**, select **GitHub Actions**
3. The workflow runs automatically — your site will be live within ~30 seconds

### Subsequent deploys

Every push to `main` triggers the workflow automatically:

```bash
git add .
git commit -m "your message"
git push
```

### Manual trigger

You can also trigger a deploy manually from **Actions → Deploy to GitHub Pages → Run workflow**.

---

## GitHub Actions workflow

The workflow (`.github/workflows/deploy.yml`) runs on every push to `main`:

1. Checks out the repository
2. Configures the GitHub Pages environment
3. Uploads the repo root as a Pages artifact
4. Deploys via `actions/deploy-pages@v4`

No build step. The entire repo root (excluding `.github/`) is served as-is.

Required repository permissions (set automatically by the workflow):
- `pages: write`
- `id-token: write`

---

## Troubleshooting

**Table doesn't load / "proxy failed" error**
- The CORS proxies are free public services and may be rate-limited or temporarily unavailable. Wait a moment and click **רענן** (Refresh).
- Some browser extensions (ad-blockers, privacy tools) may block requests to proxy domains. Try disabling them or opening in a private window.

**Data looks stale**
- The `lastUpdate` badge in the header shows the date of the underlying fund data. funder.co.il updates the data on trading days — weekends and holidays will show the last trading day.

**Clipboard copy doesn't work**
- The Clipboard API requires either HTTPS or `localhost`. Opening `index.html` as a `file://` URL disables it. Use the local server option above, or the live GitHub Pages URL.

**GitHub Actions deploy fails**
- Confirm **Settings → Pages → Source** is set to **GitHub Actions** (not a branch).
- Check the Actions tab for the specific error — the most common cause is missing Pages permissions on a freshly created repo.

---

## Disclaimer

Data is sourced from funder.co.il and displayed for informational purposes only. This is not investment advice.
