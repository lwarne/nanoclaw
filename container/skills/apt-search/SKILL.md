---
name: apt-search
description: Manage the apartment listing search — run ad-hoc searches, change search criteria (city, price, bedrooms, keywords), view/reset listings, and control the daily digest schedule. Use whenever the user asks about apartments, housing, listing results, or wants to tweak the search setup.
allowed-tools: Bash
---

# Apartment Search Tool

Automated daily apartment search powered by Craigslist HTML scraping. All persistent state lives in `/workspace/group/`.

## Files

| File | Purpose |
|------|---------|
| `/workspace/group/apt-config.json` | Search criteria — edit to change anything |
| `/workspace/group/apt-fetch.sh` | Fetch + filter + dedup script |
| `/workspace/group/apt-seen.json` | Array of listing IDs already reported (dedup state) |
| `/workspace/group/apt-new.json` | Most recent new listings (output of last run) |

---

## Run a search right now

```bash
bash /workspace/group/apt-fetch.sh
```

- Exit 0 → new listings written to `apt-new.json`
- Exit 1 → nothing new since last run
- Exit 2 → fetch/parse error

Then read the results:

```bash
cat /workspace/group/apt-new.json
```

Or pretty-print:

```bash
python3 -c "
import sys, json
for l in json.load(open('/workspace/group/apt-new.json')):
    print(f\"\${l['price']:,} | {l['neighborhood']} | {l['title'][:60]}\")
    print(f\"  {l['url']}\")
"
```

---

## Changing search criteria

Edit `/workspace/group/apt-config.json` directly, then run the script to test.

### Full config reference

```json
{
  "city":             "sfbay",
  "min_price":        10000,
  "max_price":        15000,
  "min_bedrooms":     4,
  "max_bedrooms":     0,
  "keywords_exclude": [],
  "report_time":      "08:00",
  "max_age_days":     7
}
```

| Field | Notes |
|-------|-------|
| `city` | Craigslist subdomain (see table below) |
| `min_price` / `max_price` | Monthly rent in dollars. Price filter runs locally (Craigslist URL params are broken) |
| `min_bedrooms` | Passed as URL param — Craigslist applies it server-side |
| `max_bedrooms` | Also passed as URL param; `0` means no upper limit |
| `keywords_exclude` | Case-insensitive substring matches against listing title |
| `report_time` | For human reference only — the actual schedule is set in the task (see below) |
| `max_age_days` | Not yet used — dedup handles freshness implicitly (re-run daily = naturally recent) |

### City codes

| Metro | Code |
|-------|------|
| SF Bay Area | `sfbay` |
| New York | `newyork` |
| Los Angeles | `losangeles` |
| Chicago | `chicago` |
| Seattle | `seattle` |
| Austin | `austin` |
| Boston | `boston` |
| Dallas | `dallas` |
| Denver | `denver` |
| Miami | `miami` |
| Portland | `portland` |

---

## Managing the daily scheduled task

Use the built-in MCP tools — no manual file writes needed.

### See current tasks

Use the `list_tasks` tool. It shows task IDs, schedule, status, and next run time.

### Pause the daily task

Use the `pause_task` tool with the task ID from `list_tasks`.

### Resume a paused task

Use the `resume_task` tool.

### Cancel (permanently delete) a task

Use the `cancel_task` tool.

### Change the schedule

Cancel the existing task, then create a new one with `schedule_task`:
- `schedule_type`: `"cron"`
- `schedule_value`: cron expression (in **local timezone**, handled automatically)
- `context_mode`: `"isolated"` (search runs independently, no chat history needed)
- Include the full task prompt (see below)

### Cron examples

```
"0 8 * * *"    → 8:00am daily
"0 15 * * *"   → 3:00pm daily
"0 8 * * 1-5"  → 8:00am weekdays only
"0 8 * * 1"    → 8:00am Mondays only
"0 8,20 * * *" → 8:00am and 8:00pm daily
```

### Standard task prompt

Use this when recreating the task:

```
Run the apartment listing check:
1. Execute: bash /workspace/group/apt-fetch.sh
2. If exit code 1 (nothing new): use send_message to send "No new listings today."
3. If exit code 0: read /workspace/group/apt-new.json and use send_message to send a
   concise WhatsApp-formatted report. For each listing include: price, neighborhood,
   title, and URL. Group by price range. Flag anything under $11,000 as "Great value".
   Keep report under 1000 chars.
```

Customize the `$11,000` threshold and format as desired when recreating.

---

## Reset seen listings (get a fresh batch)

```bash
echo '[]' > /workspace/group/apt-seen.json
```

Then run the fetch — all current in-range listings will appear as new.

---

## How apt-fetch.sh works

The script is a bash wrapper around an inline Python block (`python3 - "$SCRIPT_DIR" << 'PYEOF' ... PYEOF`). All logic is in Python.

### Why HTML scraping (not the JSON API)?

Craigslist's `?format=json` endpoint broke in early 2026. Their server now returns a 301 redirect that strips the `format=json`, `min_ask`, and `max_ask` parameters. The HTML page still contains all listing data in `<li class="cl-static-search-result">` elements.

### Fetch flow

1. Reads `apt-config.json`
2. Builds URL: `https://{city}.craigslist.org/search/apa?min_bedrooms={n}`
3. Fetches with Python `urllib` + `CookieJar` (handles the cookie redirect automatically)
4. Parses HTML via regex for `cl-static-search-result` elements: extracts ID (from URL slug), title, price, URL, neighborhood
5. Filters by price range in Python (URL params don't work for price)
6. Applies keyword exclusions (title substring match)
7. Loads `apt-seen.json`, diffs, writes `apt-new.json`
8. Appends new IDs to `apt-seen.json` (keeps last 2000)

### Modifying the script

**Add a new output field** (e.g., postal code): extract it from the HTML element and add it to the `listings.append({...})` dict in the Python block.

**Change data source** (e.g., Zillow, Apartments.com): replace the `urllib.request.open(...)` + `listing_re.findall(html)` section. The dedup logic (`seen_set`, `apt-new.json`, `apt-seen.json`) stays the same.

**Craigslist HTML breaks**: use `agent-browser` as a fallback — it renders JS and can extract the same data from the page.

### Testing a change

After editing the script, run it with an empty seen file:

```bash
echo '[]' > /workspace/group/apt-seen.json
bash /workspace/group/apt-fetch.sh
echo "Exit: $?"
cat /workspace/group/apt-new.json | python3 -c "import sys,json; data=json.load(sys.stdin); print(f'{len(data)} listings')"
```
