# Setup notes

Specifics that don't fit cleanly in the README's Quickstart. Read this before activating in production.

## SerpAPI account setup

[SerpAPI](https://serpapi.com/) wraps Google search (plus Bing, Yahoo, DuckDuckGo, and others) in a clean JSON API so you don't have to scrape and don't have to worry about CAPTCHAs. It's the simplest way to monitor mentions across review sites in a single workflow.

### Plans, in plain numbers

| Plan | Price | Searches/month |
|---|---|---|
| Free | 0 USD | 100 |
| Developer | 50 USD | 5,000 |
| Production | 150 USD | 15,000 |
| Big Data | 275 USD | 30,000 |

Sign up at [serpapi.com](https://serpapi.com/), confirm your email, and grab the API key from the dashboard.

### Polling cadence math, the part that decides which plan you need

This workflow makes **2 SerpAPI searches per cycle** (one Google review query, one Reddit query). Multiply by your cadence:

| Cadence | Cycles/day | Searches/day | Searches/month |
|---|---|---|---|
| Every 30 min | 48 | 96 | ~2,880 |
| Every hour | 24 | 48 | ~1,440 |
| Every 2 hours | 12 | 24 | ~720 |
| Every 6 hours (default) | 4 | 8 | ~240 |
| Every 12 hours | 2 | 4 | ~120 |
| Every 24 hours | 1 | 2 | ~60 |

**The honest read**: the SerpAPI free tier (100/month) does not support continuous review monitoring at any cadence faster than once a day. Even the every-12-hours cadence (~120/month) blows the free tier by 20%.

**Practical decisions**:

- **Free tier only, willing to live with daily checks**: lengthen the Schedule trigger to every 24 hours, expect ~60 searches/month, comfortable inside the free 100.
- **Free tier only, want faster reaction**: don't use SerpAPI. Use Google CSE or Bing Web Search API (see below).
- **Paying for a Developer plan (50 USD/month)**: every 30 minutes is comfortable inside 5,000 searches/month, with ~2,000 searches of headroom for retries and additional platforms.
- **Anything tighter than 30 minutes**: wasteful. Google search results don't refresh that often, you're paying for the same results twice.

The default in this template is **every 6 hours** because that's the slowest cadence where "find out within the hour" is still roughly true (worst case is 6 hours, average case is 3). It overspends the free tier 2.4x but fits comfortably in Developer.

### Alternative search providers if SerpAPI is too expensive

If 50 USD/month for SerpAPI is too much for the value, these are the realistic substitutes:

**Google Programmable Search Engine (CSE) + Custom Search JSON API**
- 100 free searches/day (3,000/month), then 5 USD per 1,000 searches up to 10,000/day
- Same effective coverage as SerpAPI for site-restricted queries (`site:trustpilot.com`, `site:reddit.com`, etc.)
- Setup is more work: you create a CSE in Google Cloud, enable the Custom Search JSON API, get an API key, and configure the CSE to search the entire web (default is your sites only)
- Swap the SerpAPI HTTP Request nodes for Google CSE nodes pointing at `https://www.googleapis.com/customsearch/v1?key=<KEY>&cx=<CSE_ID>&q=<QUERY>`

**Bing Web Search API (via Microsoft Azure)**
- 1,000 free searches/month on the Free F1 tier, then 3 USD per 1,000 on the S1 tier
- Different result set than Google (Bing has its own crawl), so coverage of niche review sites varies
- Endpoint: `https://api.bing.microsoft.com/v7.0/search?q=<QUERY>` with the `Ocp-Apim-Subscription-Key` header

**Direct platform APIs (no search engine in the loop)**
- **Google My Business API**: free, requires verified business ownership of the location, returns reviews directly. Best option if you only care about Google reviews on your own listing.
- **App Store Connect Customer Reviews API**: free, returns iOS reviews directly. Even better, the public iTunes RSS feed works without auth: `https://itunes.apple.com/<country>/rss/customerreviews/id=<APP_ID>/json`.
- **Trustpilot Business API**: requires a paid Trustpilot Business account. Returns reviews directly.
- **Reddit JSON API**: free with a registered app. `https://www.reddit.com/search.json?q=<QUERY>` returns mentions.

The trade-off is one-API-for-everything (SerpAPI) vs N-APIs-for-N-platforms (direct). For one or two platforms, direct is cheaper and faster. For four or more, SerpAPI's normalization saves enough engineering time to justify the cost.

## Search query patterns per platform

The default `Config Brand Settings` node ships with two queries:

```
review_query: "BRAND" review site:google.com OR site:trustpilot.com OR site:g2.com
reddit_query: BRAND site:reddit.com
```

These are deliberate but not always optimal. Adjust per your situation:

### Google My Business reviews

The most precise query, if you know your place id:

```
"BRAND" review site:google.com/maps
```

Or use the SerpAPI `google_maps_reviews` engine directly (different SerpAPI endpoint, different parameters):

```
https://serpapi.com/search?engine=google_maps_reviews&place_id=<YOUR_PLACE_ID>&api_key=<KEY>
```

This returns structured review records with rating, author, and timestamp instead of search snippets, which is much better for sentiment classification because you have the actual review text instead of a 160-character preview.

### Trustpilot

```
"BRAND" site:trustpilot.com
```

For more recent results add `inurl:reviews`. Trustpilot's URL pattern is `trustpilot.com/review/<domain>`, so `inurl:reviews` filters out their company landing pages.

### App Store

SerpAPI returns App Store search results but the snippets are usually app descriptions, not reviews. The iTunes RSS feed is far better for this:

```
https://itunes.apple.com/us/rss/customerreviews/id=<APP_ID>/sortBy=mostRecent/json
```

Replace `us` with the country code you care about. The feed returns up to 50 reviews, paginated by adding `/page=2/`, `/page=3/`, etc. No API key, no rate limits worth worrying about. Use a separate HTTP Request node for this and merge the results into the same downstream pipeline.

### Reddit

```
BRAND site:reddit.com
```

The default. Reddit's own search is famously bad, so going through Google (via SerpAPI) gives better recall. If you want to filter by subreddit:

```
BRAND site:reddit.com/r/<subreddit>
```

Reddit's JSON API is also free if you'd rather skip the search-engine middleman:

```
https://www.reddit.com/search.json?q=BRAND&sort=new&t=day
```

User-Agent header required (anything non-empty works), no auth needed for read-only search.

### Disambiguating common brand names

If your brand name is a common word ("Apex", "Pulse", "Vento", etc.), the default queries return mostly noise. Add disambiguators to the `Config Brand Settings` expressions:

```
"Vento" event app review site:google.com OR site:trustpilot.com
"Apex" -basketball -legends review site:google.com OR site:trustpilot.com
```

Test the queries directly on Google first. If the top 10 results are about your product, the query is good enough. If half are about something else, tighten until they aren't.

## Google Sheets schema for the review log

Create a single spreadsheet, one tab named **`Reviews`**, with these column headers in row 1:

| A | B | C | D | E | F | G |
|---|---|---|---|---|---|---|
| Checked At | Source | Title | Sentiment | Severity | Summary | Link |

The `Log to Google Sheets` node is configured to write into these exact column names. If you rename any of them, update the node's column mapping accordingly.

The spreadsheet id is the long string in the URL between `/d/` and `/edit`:

```
https://docs.google.com/spreadsheets/d/1AbC...XYZ/edit#gid=0
                                         ^^^^^^^^^^^
                                         this is the id
```

### Optional second tab: trends

If you want a weekly trend view, add a second tab named `Weekly Trends` with these columns and a formula like:

```
A: Week of   (=DATE column rounded to Monday)
B: Mentions  (=COUNTIF on Reviews tab)
C: Negative %  (=COUNTIFS where Sentiment="negative" / Mentions)
D: Avg Severity  (=AVERAGEIF, mapped low=1/medium=2/high=3)
```

You don't need a separate workflow to populate it, just write the formulas once and they update as the `Reviews` tab grows.

## Airtable schema, if you'd rather use Airtable

Same shape, different tool. Create a base, one table named **`Reviews`**, fields:

| Field name | Type |
|---|---|
| Checked At | Date (with time) |
| Source | Single line text |
| Title | Single line text |
| Sentiment | Single select: positive, neutral, negative, mixed |
| Severity | Single select: low, medium, high |
| Summary | Long text |
| Link | URL |

Replace the `Log to Google Sheets` node with an Airtable node (Append/Create), point it at this base/table, and map the fields by name. The downstream branch (Slack alert) doesn't change.

Airtable's Single Select fields make the trend view a lot easier to build inside Airtable itself, you get filtered views and grouped counts without writing formulas. Sheets is simpler to share and faster to set up, Airtable is better for the medium-term trend view.

## Workflow timezone

The Schedule trigger fires in n8n's instance timezone, not the workflow's. If your container runs in UTC and your cadence is "every 6 hours starting at 00:00", you get cycles at 00, 06, 12, 18 UTC. For Cairo (UTC+3, no DST), that's 03, 09, 15, 21 local, which is fine for review monitoring (the cycle that fires while you're asleep gives you the morning's alerts queued up).

If you want cycles at human-friendly local times, set `TZ` and `GENERIC_TIMEZONE` on the n8n instance:

```bash
environment:
  - TZ=Africa/Cairo
  - GENERIC_TIMEZONE=Africa/Cairo
```

Restart n8n. The Schedule trigger's "next run" indicator now shows local time.

## Anthropic spend cap

Before you activate, set a console-level spend cap on the Anthropic API key:

1. [console.anthropic.com](https://console.anthropic.com/) -> Settings -> Limits
2. Set monthly spend cap to 5 USD (more than 5x the realistic worst case for this workflow)
3. Set per-minute rate limit to something modest (60 RPM is plenty)

This is two minutes of work that prevents a runaway-cron scenario from costing you anything serious. Even at every-30-minute cadence with 30 results per cycle, monthly Claude cost stays under 2 USD.

## Activating in production

Once you've done the above:

1. **Execute Workflow** (manual run) and confirm:
   - Both SerpAPI nodes return JSON with an `organic_results` array (it can be empty if your brand had no Google hits in the past day, that's fine)
   - The Code node merges both
   - Each result gets a sensible JSON sentiment classification
   - Anything not `positive` produces a Slack message
   - Every classified result lands as a new row in the Sheet
2. Open the Schedule trigger node and **Activate** the workflow.
3. Wait one full cycle. Confirm a real cycle (not your manual run) shows up green in the Executions panel and writes to the Sheet.
4. Add a calendar reminder for a week from now to look at the Sheet and read the classifications. The first week is for tuning the prompt and the queries, not for trusting the alerts blindly.
