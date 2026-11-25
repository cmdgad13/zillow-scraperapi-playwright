# Zillow scraper that survives: why a direct Playwright run dies in a few dozen requests, what each rendered page costs in credits, and which plan is actually enough (full plan table + the 984-listing ceiling)

A direct Playwright run against Zillow dies fast. Public write-ups on this put the wall at roughly 15–30 requests before CAPTCHA pages, empty responses, or flat 403s start, and the usual stealth tricks (patched headless flags, randomized viewports, fake mouse movement) buy you about double that. The bottleneck sits at the IP layer, which is why Zillow scraping projects end up routing traffic through a proxy and rendering service such as ScraperAPI.

The cost side matters just as much. ScraperAPI bills per request in credits, and a standard page is 1 credit, but Zillow needs JavaScript execution: `render=true` is **10 credits per request** on the official rate card. Bot-protection bypass adds another 10 when it triggers, premium proxies another 10. Budget **10 to 25 credits per Zillow page**, and confirm the real number with the Domain Cost Estimator in your dashboard before you size a plan. A 100,000-credit Hobby month is 10,000 rendered pages at the 10-credit rate, which sounds generous until you start pulling detail pages.

Below: the full plan ladder, where Zillow's data actually lives, why pagination can't get you a whole city, and how the per-thousand-record math compares with managed Zillow scrapers.

> **Quick read for most people:** test your parser on the **7-day trial (5,000 credits, no credit card)** and stop there if you only need a sample. Running daily refreshes on one metro's list pages fits **Hobby (100,000 credits ≈ 10,000 rendered pages)**. Add detail pages for every listing, or cover several cities, and you're in **Startup (1,000,000 credits)** territory. 👉 [See the current plan ladder and credit allowances](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## All ScraperAPI plans, side by side

Everything ScraperAPI currently publishes, nothing trimmed. Prices sit behind the pricing page link in the last column because the tiers are priced per credit allowance and shift with billing cycle and annual terms.

| Plan | API credits / month | Concurrent threads | Geotargeting | Price and signup |
| --- | --- | --- | --- | --- |
| Free | 1,000 | 5 | Not listed | [ Current price](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Hobby | 100,000 | 20 | US & EU | [ Current price](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | 1,000,000 | 50 | US & EU | [ Current price](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | 3,000,000 | 100 | Country-level | [ Current price](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 | Country-level | [ Current price](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | 10,500,000 | 300 | Country-level | [ Current price](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | 21,500,000 | 500 | Country-level | [ Current price](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 500+ | Country-level | [ Current price](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

Two details in that table change how you read it. Hobby and Startup both cap geotargeting at US & EU, so if your Zillow work needs per-country targeting beyond that, the jump is to Business at 3,000,000 credits, not Startup. And credit volume scales far faster than concurrency: Hobby to Startup is 10x the credits but only 2.5x the threads.

The billing mechanics that matter before you commit:

- **7-day trial, 5,000 credits, no credit card.** That is 500 rendered pages at the 10-credit rate, enough to validate a parser against live Zillow.
- **Free plan after the trial:** 1,000 credits a month, 5 concurrent threads.
- **Hitting your concurrency cap returns a 429.** The request is rejected, not queued, and you aren't charged extra for trying.
- **Running out of credits pauses scraping** until renewal. Auto-upgrade to the next tier is opt-in on Hobby, Startup, Business, Scaling, and Professional. Pay-as-you-go past your allowance, with a monthly cap you set, is available on Scaling, Professional, Advanced, and Enterprise.
- **7-day refund, no questions asked**, if the service doesn't work out.
- **Enterprise is custom-quoted** above 22M credits/month; sales handles it directly.

## What a Zillow page actually costs in credits

Zillow is not a plain HTML fetch. The rate card below is the official per-request cost, and the add-ons stack.

| Request type | Credits per request |
| --- | --- |
| Standard page | 1 |
| JS rendering (`render=true`) | 10 |
| Bot-protection bypass (Cloudflare, DataDome, PerimeterX/HUMAN, Akamai) | 10 |
| Premium proxies (`premium=true`) | 10 |
| Premium proxies + JS rendering | 25 |
| Ultra premium (`ultra_premium=true`) | 30 |
| Ultra premium + JS rendering | 75 |
| E-commerce (Amazon, Walmart, eBay) | 5 |
| SERP (Google, Bing) | 25 |
| LinkedIn | 30 |
| Geotargeting, sessions, device type, output format, auto-parsing | 0 |

The 10-credit figure you'll see quoted for Zillow pages is the rendering surcharge alone. If a request also trips bot-protection bypass, you're at 20; premium proxies plus rendering is 25. Every response carries an `sa-credit-cost` header showing what that request cost, and you can set `max_cost` per scrape so a single expensive URL can't quietly drain a month of credits.

This is the number to build a budget on, because the split between list pages and detail pages is lopsided. Pulling the full 984 listings from one search area takes about 24 page loads at roughly 41 cards per load, so call it 240 credits. Fetching the detail page for each of those 984 homes costs 9,840 credits. Detail pages run about 40x the list layer for the same inventory. 👉 [Check the current credit rates against your page volume](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Where Zillow's data hides, and the parser you still have to write

Zillow renders listing data client-side and embeds it in a `__NEXT_DATA__` JSON blob in the page source. Reading that script tag beats chasing CSS selectors, which break whenever Zillow reshuffles the DOM. On a single property page, the details sit under the `gdpClientCache` key inside the same blob. Map-box searches carry a `searchQueryState` parameter, which is URL-encoded JSON describing the bounding box and filters.

ScraperAPI returns the HTML after rendering. It does not hand you Zillow fields. There is no Zillow-specific parser or structured output documented for it, so the `__NEXT_DATA__` extraction, the schema checks, and the graceful failure when Zillow renames a key are your code. Budget a few days of engineering, plus ongoing maintenance, on top of the credit spend. That labor is the real reason the per-thousand-record comparison further down isn't apples to apples.

## The 984-listing ceiling: why pagination can't cover a city

Zillow stops returning results at **984 listings per search**. Ask for more and the count doesn't move. Sorting applies within a search area only, and once you split an area, the page counter resets to 1 in each new region.

So "scrape all of Austin" is a geometry problem, not a pagination problem. You divide the map into tiles, check each tile's result count, and subdivide any tile that lands at or near the cap until every tile sits comfortably under 984. Sorting by price or newest inside a tile doesn't help you escape the cap, because the cap applies per search area, not per sort.

The same constraint shows up in every serious tool built on this site. One Apify actor splits the map into four quadrants recursively, with a `maxLevel` default of 5, to defeat Zillow's official limit of 1,000 API calls per day and 20 calls per page. Another lists a single-search ceiling of 820 records in its no-zoom mode. Tiling is the workaround everyone converges on.

For a scheduled version of that workflow, ScraperAPI's Crawler takes a `start_url`, a `crawl_budget` denominated in credits, a `max_depth`, include/exclude regexes, and a webhook callback, with intervals of once, hourly, daily, weekly, or monthly. The documentation's own sample job uses a Zillow search URL as the start page with `country_code` set to `us`. If you want to crawl listing pages and follow through to detail pages, that's the piece to read before writing your own scheduler.

## Wiring it into Playwright

The proxy endpoint takes a standard Playwright proxy config, and each request exits through a fresh residential IP:

python
browser = p.chromium.launch(
    headless=True,
    proxy={
        "server": "http://proxy-server.scraperapi.com:8001",
        "username": "scraperapi",
        "password": SCRAPER_API_KEY,
    }
)


Passing `country_code` routes the request through the residential pool for that country, which is what you want for US Zillow inventory. Keep the residential routing on and skip the free-proxy-rotation route entirely; datacenter IP ranges get flagged by Zillow almost immediately, and hand-rotating free proxies burns more time than the credits cost.

## ScraperAPI vs managed Zillow scrapers: the per-thousand math

These are different products. ScraperAPI sells request infrastructure (proxies, rendering, retries). Apify's actors sell finished Zillow records with fields already parsed. Compare them on total cost, which includes your parsing time.

| Option | Published price | What you get | Notable limit |
| --- | --- | --- | --- |
| ScraperAPI | Credit-based; Zillow page = 10–25 credits depending on add-ons | Rendered HTML from residential IPs; you write the parser | No Zillow field parser included |
| Apify, one Zillow actor (Dec 2023 listing) | $2 per 1,000 results | Parsed listings, JSON/CSV/Excel/HTML out | 820 records per search in its no-zoom mode |
| Apify, another Zillow actor (fields verified Sep 2026) | $3.10/1,000 on Free, down to $1.20/1,000 on Business, plus $0.001 per run start | Parsed listings with filters, agent contacts, price and tax history | 10,000 rows per run; 2,000 Austin listings in 47 seconds in its test |
| Apify, internal-API actor (Apache 2.0, 62 stars) | Around $0.25 per 2,000 results | Parsed records via Zillow's internal API, map-split logic built in | 200 items per run by default; allow a minute per run to amortize startup |
| Oxylabs Zillow Scraper API | No public price in its docs | 500 listings per response, batch submission of up to 5,000 URLs, `geo_location` per request | Parse step still yours unless you use its custom parser |

Read that table as a build-versus-buy decision, not a price ranking. The $0.25-per-2,000 figure looks unbeatable until you notice it's an internal-API actor with a default 200-item cap and default `maxLevel` of 5, meaning you tune depth and volume yourself. ScraperAPI's per-credit cost looks high per record until you remember it isn't selling records. If your team already has a Zillow parser and needs clean IPs at scale across multiple sites, the infrastructure pricing is the right shape. If you need a Zillow DataFrame by Friday and have no parser, a managed actor is cheaper than your engineering hours.

## The legal boundary

hiQ Labs v. LinkedIn (Ninth Circuit, 2022) held that scraping publicly accessible pages doesn't violate the Computer Fraud and Abuse Act. That covers the access question in the US, not Zillow's terms of service, which are a separate contract issue.

Personal data is the sharper edge. Agent names, phone numbers, and email addresses are personal data, and GDPR can apply depending on where you and the data subjects are. Zillow's scale is why this keeps coming up: the platform holds 110M+ properties and draws north of 200 million monthly visits by Similarweb's count. Have a documented lawful basis before you store contact records, and don't treat the CFAA ruling as blanket permission.

## Which plan, by situation

- **Validating a parser, or pulling a single metro once:** use the trial. 5,000 credits covers 500 rendered pages, and the 984-listing ceiling means one search area never needs more than ~240 credits anyway. Don't pay for a plan yet.
- **Daily refresh of one metro's list pages:** Hobby. 100,000 credits is 10,000 rendered pages a month, and the list layer for 984 listings is about 240 credits a pass.
- **Detail pages for every listing, or several cities:** Startup at 1,000,000 credits. Detail pages are what blow up budgets; 10,000 of them at 10 credits each consumes an entire Hobby month in one pass.
- **Many parallel workers:** Business at 100 threads, Scaling at 200. Concurrency, not credits, is what you're buying here.
- **Beyond country-level geotargeting:** Business is the first tier that offers it, so skip Startup even though its credit count looks like the natural step up.

Two things to confirm before you enter a card: run the Domain Cost Estimator on one real Zillow property URL and note the exact credit cost of that request, then multiply by your monthly page count (detail pages first, since they dominate). If the total lands within about 20% of a tier's allowance, take the next tier up, because auto-upgrade billing mid-month is harder to plan around than a slightly larger plan.

👉 [Start the 7-day trial with 5,000 credits and no card](https://www.scraperapi.com/?fp_ref=coupons)
