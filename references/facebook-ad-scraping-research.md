# Facebook/Meta Ad Scraping: Discovery-First Research

**Goal:** Surface the best-performing Meta ads in any category WITHOUT knowing competitors upfront — then spy on their creative DNA.

**Date:** 2026-06-14

---

## The Core Insight

Meta Ad Library (`facebook.com/ads/library/`) is a public, no-login SPA. It exposes every active ad by keyword or page. The official Marketing API requires app review, ID verification, and ad account access — and can't see competitor data anyway. Everyone bypasses it by scraping the Ad Library.

Critical distinction: **performance data (spend, impressions, CTR) does NOT exist for commercial ads.** Meta only exposes this for political/social issue ads. For commercial ads, you must use **proxy signals** to infer performance.

---

## Proxy Signals for Ad Performance (Commercial Ads)

| Signal | Strength | Why It Works |
|---|---|---|
| **Longevity** | Primary | An ad running 90+ days = someone is paying to keep it alive. Nobody funds losers. |
| **Variant count per advertiser** | Primary | Multiple creative variations of the same angle = that angle is converting. High iteration = high conviction. |
| **Platform breadth** | Secondary | If an advertiser disproportionately runs on Instagram Reels vs Facebook Feed, that format is finding efficiency. |
| **Recency + active status** | Secondary | Recently started AND still active = likely scaling up. Recently stopped = likely killed. |
| **Ad count per keyword** | Discovery | Advertisers with 30+ active variations on a keyword are outspending everyone else. This is how you find who matters. |

---

## Tools Ranked by Fit for Discovery-First Workflow

### Tier 1 — Directly Fit

| Tool | Type | Why |
|---|---|---|
| **[Apify Meta Ads Scraper (pro100chok)](https://apify.com/pro100chok/meta-ads-library-scraper)** | Cloud actor | Keyword search mode, 240+ countries, all 6 ad categories, TLS fingerprint impersonation, JS challenge solving, structured JSON output. Best overall match. |
| **[Apify Facebook Ad Library Scraper (makework36)](https://apify.com/makework36/facebook-adlib-scraper)** | Cloud actor | Keyword search, pageAds mode, April 2026 update improved keyword responsiveness. Runner-up. |
| **[Apify Meta Ads Library Scraper Pro (aiscraperdev)](https://apify.com/aiscraperdev/meta-ads-library-scraper)** | Cloud actor | Hardware-level scrolling to bypass anti-bot limits. Important for deep pagination through keyword results. |

### Tier 2 — Supporting Role

| Tool | Type | Role |
|---|---|---|
| **playwright-stealth** | Python library | Foundation for custom scraping beyond what Apify covers (e.g., video transcript extraction). Playwright > Puppeteer for Meta in 2026. |
| **Residential sticky proxies** | Infra | Mandatory at scale. Rotating too fast triggers device-change detection. Keep sessions alive for minutes. |
| **FireCrawl** | Scraping API | After identifying top ads, pull their landing pages for conversion mechanism analysis. |

### Tier 3 — Limited Fit

| Tool | Why Limited |
|---|---|
| **Ad Swipe Local (Chrome extension)** | Manual tool. Useful for qualitative review, not for automated discovery across keywords. |
| **meta-ads-collector (Python CLI)** | Free but limited search modes. Won't give cross-advertiser landscape view. |
| **Ezee-Kits BY-PASS-ANTIBOT (GitHub)** | Overkill. Apify actors already handle bot detection. Only relevant if building a competing product. |

---

## Technical Approach

### How Scrapers Work Under the Hood

1. **Public SPA scraping** — hit `facebook.com/ads/library/` with browser automation, not raw HTTP
2. **GraphQL interception** — Meta's internal `AdLibrarySearchPaginationQuery` persisted queries are the real API
3. **TLS fingerprint impersonation** — Chrome TLS fingerprints via `curl_cffi` to avoid 403s
4. **JS challenge solving** — auto-solve Meta's `__rd_verify` bot-detection challenge
5. **Session persistence** — keep `userDataDir` across runs to avoid re-CAPTCHA
6. **Rate limits** — ~80-100 requests per session before throttling

### Core Scraping Pattern

```python
from playwright.async_api import async_playwright
from playwright_stealth import stealth_async

async def scrape_meta_ad_library(query: str, proxy: str):
    async with async_playwright() as p:
        browser = await p.chromium.launch(proxy={"server": proxy})
        page = await browser.new_page()
        await stealth_async(page)
        await page.goto(
            f"https://www.facebook.com/ads/library/"
            f"?active_status=active&ad_type=all&q={query}",
            wait_until="networkidle"
        )
        # scroll-load ad cards, extract from React DOM
        await browser.close()
```

---

## Recommended Pipeline: Discovery-First

```
1. INPUT: Keywords/categories to investigate (NOT competitor URLs)
       ↓
2. DISCOVER: Apify keyword search across ALL advertisers
   → Returns: ads from all advertisers matching keyword, with
     ad_id, page_name, page_id, body, headline, CTA, media,
     landing_page, start_date, active_status, platforms
       ↓
3. SCORE (Two levels):
   AD-LEVEL:
     • Longevity: days active (start_date → today if active)
     • Platform breadth: more platforms = higher confidence
   ADVERTISER-LEVEL:
     • Ad count for this keyword
     • Average longevity across their ads
   → Output: "These N advertisers dominate this space.
     Here are their longest-running ads, ranked."
       ↓
4. DEEP-DIVE: Full ad history for top-N advertisers
   (pageAds mode using discovered page IDs from step 3)
   → Find: variant patterns, creative iteration velocity,
     full funnel structure (which ads link to which LPs)
       ↓
5. EXTRACT Creative DNA:
   • Body text → NLP: hooks, pain points, offers, CTAs
   • Media → AI vision: format (UGC/studio/motion),
     colors, text overlay patterns
   • Landing pages → structure: headline patterns,
     social proof types, pricing presentation
       ↓
6. OUTPUT: Ranked winning ads + creative brief of patterns
```

---

## Platform Comparison (2026)

| Platform | Auth Required | Rendering | Rate Limit | Proxy Type |
|---|---|---|---|---|
| **Meta Ad Library** | No | React SPA | ~80-100 req/session | Residential sticky |
| **Google Ads Transparency** | No | Angular SPA | ~200-300 req/hr | Residential sticky |
| **TikTok Creative Center** | No | API-based | ~1,000+ req/hr | Datacenter OK |
| **TikTok Ad Library** | Yes (login) | React SPA | Unknown | Residential rotating |

---

## Critical Limitations

1. **No spend/impression data for commercial ads** — Meta policy, not a tool limitation
2. **Media URLs are temporary CDN links** — download promptly after scraping
3. **Inactive commercial ads only kept ~1 year** (political ads: up to 7 years)
4. **Results capped without proxies** — Meta rate-limits aggressively per IP
5. **Legal gray area** — public data scraping generally legal but may violate Meta ToS for commercial use

---

## Open Questions (Not Answered by Current Research)

1. How many unique advertisers does a keyword search actually return? Is there a ceiling?
2. Can you filter by media type (video-only) in keyword search mode reliably?
3. How far back does keyword search reach for inactive commercial ads?
4. What's the actual pagination depth limit on keyword searches?

---

## Key Twitter Sources

- [@mikefutia](https://x.com/mikefutia) — Claude Code creative strategist scraping competitor Facebook pages
- [@codyschneider](https://x.com/codyschneider) — Bulk ad generator + n8n automation for Facebook Ads
- [@adrian_horning_](https://x.com/adrian_horning_) — Facebook Ad Library scraper via scrape creators
- [@0xROAS](https://x.com/0xROAS) — Ad library downloads + AI video cloning workflows
- [@ivesparrowai](https://x.com/ivesparrowai) — AI agent for Meta Ads Library + TikTok scraping
- [@alecsandrull](https://x.com/alecsandrull) — Competitor strategy extraction from Facebook Ad Library
- [@knoxtwts](https://x.com/knoxtwts) — Longevity-based winner identification from Ad Library

---

## References

- [Apify Meta Ads Library Scraper (pro100chok)](https://apify.com/pro100chok/meta-ads-library-scraper)
- [Apify Facebook Ad Library Scraper (makework36)](https://apify.com/makework36/facebook-adlib-scraper)
- [Apify Meta Ads Library Scraper Pro (aiscraperdev)](https://apify.com/aiscraperdev/meta-ads-library-scraper)
- [Ad Swipe Local Chrome Extension](https://chromewebstore.google.com/detail/ad-swipe-local-%E2%80%93-save-met/eldlibkbjpnjbakhikifbopmdoghaihh)
- [meta-ads-collector (Python)](https://www.piwheels.org/project/meta-ads-collector/)
- [DataResearchTools: Scraping Competitor Ad Libraries 2026](https://dataresearchtools.com/scraping-competitor-ad-libraries-meta-google-tiktok-in-2026/#content)
