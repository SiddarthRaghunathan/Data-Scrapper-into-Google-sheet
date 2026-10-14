# TotalCorner → Google Sheets Scraper

> Production scraper that pulls **Full-Time football match data** from TotalCorner.com and incrementally syncs 150k+ rows into Google Sheets — bypassing Cloudflare with Playwright.

[[Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/SiddarthRaghunathan/Data-Scrapper-into-Google-sheet/blob/main/Total_Corner_Schedule_Page_Scapper.ipynb)
[[Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[[Playwright](https://img.shields.io/badge/Playwright-Chromium-45ba4b?logo=playwright)](https://playwright.dev/)
[[License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**Notebook:** `Total_Corner_Schedule_Page_Scapper.ipynb`  
**Sheet:** `Total Corner Sandbox`  
**Source:** `https://www.totalcorner.com/match/schedule/{YYYYMMDD}/page:{N}`

---

### What it does

1.  **Bypasses Cloudflare** using Playwright Chromium with realistic UA, locale, timezone
2.  **Scrapes** every match block (`data-match_id`) — 359 matches per page
3.  **Parses** League, Home/Away, League Positions `[12]`, Handicap, FT Score, Corners `6 - 1 (2 - 0)`, Dangerous Attacks `45 - 17`, Red/Yellow Cards
4.  **Filters** out youth/women/reserves: `U20|Women|U23|U19|Reserves|Friendlies`
5.  **Incrementally loads** into Google Sheets — dedupes by `MatchId`, only adds new rows
6.  **Current scale:** 159,401 rows after last run (+3366 new rows per sync)

### Data Schema

| Column | Example | Notes |
| :--- | :--- | :--- |
| DateID | 20260301 | YYYYMMDD from URL |
| League | Ecuador LigaPro Serie A | `td_league` |
| MatchId | 189905931 | `data-match_id` |
| Home / Away | Emelec / Delfin SC | cleaned of `[Pos]` and cards |
| HomeLeaguePosition | 15 | extracted from `[15]` |
| Handicap | -0.75 | |
| Score | 1 - 1 (0 - 0) | FT |
| Corners | 6 - 1 (2 - 0) | Full (HT) |
| DangerousAttack | 45 - 17 (24 - 12) | Full (HT) |
| HomeRedCard / HomeYellowCard | 1 / 5 | parsed from prefix numbers |
| AwayRedCard / AwayYellowCard | NaN / 4 | parsed from suffix numbers |

### Why Playwright?

TotalCorner uses Cloudflare Turnstile. `requests` returns `just a moment / cf-challenge` page. Solution:

```python
async with async_playwright() as p:
    browser = await p.chromium.launch(headless=True)
    context = await browser.new_context(
        user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36...",
        locale="en-US",
        timezone_id="Europe/Dublin",
        viewport={"width": 1365, "height": 900}
    )
    page = await context.new_page()
    await page.goto(url, wait_until="domcontentloaded", timeout=90000)
    await page.wait_for_timeout(8000)
    html = await page.content()
```

### Core Functions

```python
# Cell 2 - Sandbox page pull (FIXED - FT detection)
parse_totalcorner_html_to_match_df(html) # → DataFrame filtered to Time == "Full"

# Cell 3 - PagePull Helper
async def pull_totalcorner_page(date: str, page_number: int, max_retries=3)
# Retries with base_delay * attempt, returns None after failures

# Cell 4 - ETL Helper
def get_most_recent_dateid(gc, sheet_name="Total Corner Sandbox")
async def pull_around_dateid(gc, manual_dateid=None, days_before=2, days_after=2, pages_per_date=3, save=True)
# Computes days_gap = yesterday - max_dateid, pulls all dates in range
```

### Installation & Usage

**1. Colab (Recommended)**
```python
# Cell 1 already does this:
%%capture
!pip -q install playwright nest_asyncio
!playwright install
!playwright install-deps

import nest_asyncio; nest_asyncio.apply()
from google.colab import auth
auth.authenticate_user()
import gspread
from google.auth import default
creds, _ = default()
gc = gspread.authorize(creds)
```

**2. Run Incremental Sync**
```python
# Get most recent date in sheet and pull till yesterday
SHEET_NAME = "Total Corner Sandbox"
sh = gc.open(SHEET_NAME)
# ... code in Cell 6 computes days_gap automatically ...

result_df = await pull_around_dateid(gc, manual_dateid=max_dateid, days_before=0, days_after=days_gap, base_delay=3)
# Output: Added 3366 new rows. Total rows: 159401
```

### Requirements

```
playwright
nest_asyncio
beautifulsoup4
pandas
numpy
gspread
gspread-dataframe
google-auth
lxml
```

### Handling Cloudflare Blocks

Your last run shows expected behavior:
```
Attempt 1/3 failed for 20261004 page 3: Exception Blocked by Cloudflare/WAF
Retrying in 3s...
Warning: 14 page(s) failed after retries
```

Fixes implemented:
- 8s wait after `domcontentloaded`
- Retry with `base_delay * attempt`
- `await asyncio.sleep(2)` between pages

Future improvement: add proxy or `playwright-stealth`.

### One-time League Cleaner (Cell 5)

```python
sh = gc.open("Total Corner Sandbox")
ws = sh.sheet1
existing_df = get_as_dataframe(ws).dropna(how="all")
mask = existing_df['League'].str.contains('U20|Women|U23|Reserves|Friendlies|U19|U21|Thailand Division 3|Spain Youth League|...', na=False)
clean_df = existing_df[~mask]
ws.clear()
set_with_dataframe(ws, clean_df)
```

### Roadmap

- [ ] Add stealth plugin to reduce WAF blocks
- [ ] Switch to `ws.batch_update()` for 150k+ row performance
- [ ] GitHub Action cron: daily 02:00 IST sync
- [ ] Add derived columns: `TotalCorners = CornerHome + CornerAway`

### Disclaimer

For educational/research purposes. Respect TotalCorner.com ToS and rate limit. The scraper already includes 2s delay between requests.

### License

MIT © [Siddarth Raghunathan](https://github.com/SiddarthRaghunathan)

---
If useful, please ⭐ the repo!
