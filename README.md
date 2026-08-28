# Zoopla Scraper: How to Scrape Zoopla Property Data Without Getting Blocked — Python Tutorial, Anti-Bot Bypass, Plan Comparison, and Everything You Need to Get Started

You want Zoopla data. Maybe you're tracking London flat prices, building a property comparison dashboard, or feeding rental figures into a spreadsheet for your next investment decision. Whatever the reason, the idea sounds simple: write a few lines of Python, hit the URL, grab the HTML.

Then reality shows up. Zoopla throws you a blank page. Or a CAPTCHA. Or a polite-but-firm 403 that doesn't even bother explaining itself. You fiddle with headers. You add a `time.sleep(2)`. You try a fresh User-Agent string. Still nothing.

This guide is for that moment. We'll cover what data you can actually pull from Zoopla, why scraping it is trickier than it looks, how to handle the anti-bot protections properly, and where a scraping API like ScraperAPI fits into the picture — with a full breakdown of its plans so you can pick the right one without overpaying.

---

**Why Zoopla Is Worth Scraping**

Zoopla is one of the UK's biggest property portals, listing hundreds of thousands of properties for sale and rent across England, Scotland, Wales, and Northern Ireland. The data sitting publicly on those listing pages is genuinely valuable:

- **Property prices** — current asking prices, price history, estimated values (Zestimates equivalent)
- **Location data** — full addresses, postcodes, map coordinates
- **Property specs** — number of bedrooms, bathrooms, reception rooms, floor area in sq ft
- **Listing metadata** — date listed, listing status, agent contact details
- **Photos and floor plans** — image URLs for each listing
- **Nearby amenities** — schools, transport links, distances

For a researcher tracking regional price trends, a developer building a comparison tool, or an investor doing due diligence on a neighborhood, this data would take weeks to collect manually. Programmatically? It should take hours.

The problem is Zoopla's anti-scraping systems are real and they've gotten more aggressive over time.

---

**What Makes Zoopla Hard to Scrape**

Zoopla runs on Next.js, which means two things. First, a lot of the data you want is embedded inside a JavaScript `<script>` tag as JSON — not visible in the raw HTML at all if you request the page with a basic `requests.get()`. Second, the search pages (the listings index pages) are dynamically rendered and won't load results without JavaScript execution.

On top of that, Zoopla actively monitors for bot-like behavior:

- **IP rate limiting** — too many requests from one IP in a short window triggers blocks
- **TLS fingerprinting** — Python's `requests` library has a recognizable TLS signature that differs from browsers
- **Browser fingerprinting** — headless browsers without proper configuration are detected
- **User-Agent and header checks** — missing or mismatched headers raise flags

The result: a scraper that works on your first test run quietly dies when you try to scale it to a few hundred properties.

---

**The Data Hidden in Zoopla's Next.js Pages**

Here's the trick that makes scraping individual Zoopla property pages significantly easier: Zoopla embeds a full JSON object containing all the property data directly inside a `<script>` tag in the HTML. You don't need to headlessly render the page to get property details — you just need the raw HTML, and then parse the JSON out of it.

A basic approach looks like this:

python
import httpx
import json
from parsel import Selector

async def get_property_data(url: str):
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36",
        "Accept-Language": "en-GB,en;q=0.9",
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    }
    async with httpx.AsyncClient(headers=headers) as client:
        response = await client.get(url)
    selector = Selector(response.text)
    # Extract JSON data embedded in Next.js __NEXT_DATA__ script tag
    raw_data = selector.css("script#__NEXT_DATA__::text").get()
    if raw_data:
        data = json.loads(raw_data)
        return data


For individual property pages, this works surprisingly well — when you can actually get a clean response. The hard part is getting that clean response at scale.

**Search pages are a different story.** Zoopla's search results won't render without JavaScript. You either need a headless browser (Playwright, Puppeteer, Selenium) or a scraping API that handles JavaScript rendering for you.

---

**Scraping Zoopla Search Pages**

For search pages, you're looking at a URL structure like:


https://www.zoopla.co.uk/for-sale/property/london/?q=london&search_source=home&pn=1


Pagination uses the `pn` parameter — increment it to move through pages. A simple loop handles that:

python
import requests
from bs4 import BeautifulSoup

API_KEY = "your_scraperapi_key"

def scrape_zoopla_search(location: str, pages: int = 5):
    results = []
    for page in range(1, pages + 1):
        target_url = f"https://www.zoopla.co.uk/for-sale/property/{location}/?q={location}&search_source=home&pn={page}"

        # Route through ScraperAPI with JS rendering
        api_url = f"http://api.scraperapi.com?api_key={API_KEY}&url={target_url}&render=true"

        response = requests.get(api_url, timeout=60)
        soup = BeautifulSoup(response.text, "html.parser")

        listings = soup.find_all("div", {"class": "dkr2t86"})
        for listing in listings:
            price = listing.find("p", {"class": "_64if862"})
            address = listing.find("address")
            description = listing.find("p", {"class": "m6hnz63"})

            results.append({
                "price": price.text if price else None,
                "address": address.text if address else None,
                "description": description.text if description else None,
            })
    return results


> **A quick note on CSS class names**: Zoopla updates its frontend fairly regularly. The class names like `dkr2t86` and `_64if862` are generated by their build system and can change. If your selectors suddenly stop working, inspect the page fresh and update them.

---

**Discovering Properties at Scale: Sitemaps**

If you want to collect *all* Zoopla listings rather than just search results for a specific location, Zoopla's sitemap system is your friend. The robots.txt file points to a central sitemap index:


https://www.zoopla.co.uk/xmlsitemap/sitemap/index.xml.gz


That index contains links to dozens of category-specific sitemaps, split into chunks of 50,000 URLs each. There are separate sitemaps for properties for sale, properties for rent, new builds, and more. Each sitemap URL looks like:


https://www.zoopla.co.uk/xmlsitemap/sitemap/for_sale_details_001.xml.gz


You parse the gzipped XML to extract individual property URLs, then scrape each one for its full data. For tracking new listings specifically, the `new_home_details` sitemaps update daily — scraping them regularly keeps your dataset fresh without re-hitting the entire site.

---

**Why Your Plain Python Scraper Will Get Blocked**

Let's be honest about what happens without anti-bot protection. A basic `requests.get()` session against Zoopla will usually:

1. Work fine for the first 10–20 requests
2. Start returning 403 or blank pages around request 30–50
3. Hit a complete block (sometimes IP-level) if you push past that

The failure modes people encounter most often:

- **IP bans** — residential ISP IPs eventually get flagged if they send too many requests
- **TLS fingerprint mismatch** — `requests` library has a recognizable TLS signature. Cloudflare and similar systems detect it
- **Missing headers** — a real browser sends 15+ headers by default; `requests` sends 3
- **Request timing** — humans have inconsistent timing; bots have consistent timing

Adding proxies helps but doesn't solve everything. You'd also need to handle IP rotation, manage proxy pools, retry failed requests, and deal with session management — all before writing a single line of actual data parsing code.

This is where a scraping API changes the calculus.

---

**Using ScraperAPI for Your Zoopla Scraper**

ScraperAPI is a proxy rendering API: you send it a URL, it handles the proxy rotation, JavaScript rendering, CAPTCHA solving, and retry logic, and sends back clean HTML. Your scraper code stays simple. You don't manage proxies, you don't worry about fingerprinting, you just parse the response.

The integration is literally one line of change. Instead of:

python
response = requests.get("https://www.zoopla.co.uk/for-sale/property/london/")


You do:

python
response = requests.get(
    "http://api.scraperapi.com",
    params={
        "api_key": "YOUR_API_KEY",
        "url": "https://www.zoopla.co.uk/for-sale/property/london/",
        "render": "true",     # Enable JS rendering for search pages
        "country_code": "gb"  # Use UK proxies — important for geo-restricted content
    }
)


That's it. The rest of your parsing code stays unchanged.

For Zoopla specifically, `render=true` is important for search pages (where listings are loaded dynamically) but optional for individual property pages (where data is embedded in the Next.js JSON). Keep that distinction in mind — JS rendering costs more credits, so don't enable it when you don't need it.

👉 [Start your free ScraperAPI trial — 5,000 credits, no credit card required](https://www.scraperapi.com/?fp_ref=coupons)

---

**ScraperAPI's Credit System: What Actually Gets Charged**

Before picking a plan, you need to understand how credits work — because the headline "100,000 credits" means very different things depending on what you're scraping.

The base cost is **1 credit per successful request** for a standard, unprotected page. But multipliers apply:

| Request Type | Credits Per Request |
| --- | --- |
| Standard page (basic HTML) | 1 |
| JavaScript rendering (`render=true`) | +10 extra (so 11 total for standard pages) |
| Premium proxies (`premium=true`) | +10 extra |
| Ultra premium (`ultra_premium=true`) | +30 extra |
| Amazon | 5 |
| Google / Bing | 25 |
| LinkedIn | 30 |
| Cloudflare-protected site bypass | +10 extra |

For Zoopla: it's not in the ultra-high-cost tier like Google or Amazon, but if you enable JavaScript rendering for search pages, each request costs around 11 credits. Individual property pages without rendering cost just 1 credit each.

**Critically**: you're only charged for successful requests (200 or 404 responses). Failed scrapes don't burn your credits.

---

**Full ScraperAPI Plan Comparison**

Here's every plan currently available, with pricing for both monthly and annual billing (annual gets you 10% off automatically):

| Plan | Monthly Price | Annual Price | API Credits/Month | Concurrent Threads | Geotargeting | Get Started |
| --- | --- | --- | --- | --- | --- | --- |
| **Free Trial** | $0 (7 days) | — | 5,000 (one-time) | 5 | Limited | [Start Free Trial](https://www.scraperapi.com/?fp_ref=coupons) |
| **Hobby** | $49/mo | $44.10/mo | 100,000 | 20 | US & EU only | [Get Hobby Plan](https://www.scraperapi.com/?fp_ref=coupons) |
| **Startup** | $149/mo | $134.10/mo | 1,000,000 | 50 | US & EU only | [Get Startup Plan](https://www.scraperapi.com/?fp_ref=coupons) |
| **Business** | $299/mo | $269.10/mo | 3,000,000 | 100 | Global | [Get Business Plan](https://www.scraperapi.com/?fp_ref=coupons) |
| **Scaling** ⭐ Most Popular | $475/mo | $427.50/mo | 5,000,000 | 200 | Global | [Get Scaling Plan](https://www.scraperapi.com/?fp_ref=coupons) |
| **Professional** | $975/mo | $877.50/mo | 10,500,000 | 300 | Global | [Get Professional Plan](https://www.scraperapi.com/?fp_ref=coupons) |
| **Advanced** | $1,975/mo | $1,777.50/mo | 21,500,000 | 500 | Global | [Get Advanced Plan](https://www.scraperapi.com/?fp_ref=coupons) |
| **Enterprise** | Custom | Custom | 22,000,000+ | 500+ | Global | [Contact Sales](https://www.scraperapi.com/?fp_ref=coupons) |

A few important details that the plan table alone doesn't tell you:

**Geotargeting is tiered.** Hobby and Startup are locked to US and EU proxies. Since Zoopla is a UK site, US/EU proxies include UK — so for basic Zoopla scraping, the Hobby or Startup plan is fine. If your project needs proxy coverage in Asia, Latin America, or other regions, you'll need at least Business.

**Pay-as-you-go only starts at Scaling.** On Hobby, Startup, and Business, if you hit your credit limit mid-month, you either upgrade or stop. From Scaling upward, you can enable PAYG overflow at a fixed rate so you're never hard-cut off.

**Credits don't roll over.** Whatever you don't use resets at renewal. Size to your actual monthly usage, not your aspirational usage.

**Unlimited analytics history** starts at Business. Hobby and Startup give you 30 days of dashboard data.

---

**Which Plan Makes Sense for a Zoopla Scraper?**

Let's run the numbers for common Zoopla scraping scenarios.

**Scenario 1 — Personal project, property research**

You want to track 2,000 listings in a specific area weekly. Individual property pages, no JS rendering needed.

- 2,000 pages × 4 weeks = 8,000 credits/month
- **Hobby plan ($49/mo)** covers this with enormous headroom — you could scale to 100,000 individual property pages before needing to upgrade.

**Scenario 2 — Property comparison tool for a small audience**

You're scraping search pages (JS rendering required) across 5 UK cities, pulling 500 listings per city per week.

- 2,500 search page requests × 11 credits each (with rendering) = 27,500 credits/week = ~110,000 credits/month
- **Hobby plan** just barely fits, but there's no overflow cushion. **Startup ($149/mo)** is the comfortable choice at this volume.

**Scenario 3 — Production real estate intelligence tool**

You're processing 50,000 properties monthly, mix of search and property pages, with JS rendering on search.

- Conservative estimate: 30,000 property pages (1 credit each) + 20,000 search pages (11 credits each) = 250,000 credits/month
- **Startup ($149/mo)** handles this comfortably within its 1,000,000 credit allowance.

**Scenario 4 — Enterprise data collection**

Millions of scrapes monthly, need global proxies and pay-as-you-go buffer.

- **Business ($299/mo)** or **Scaling ($475/mo)** depending on exact volume. At this scale, annual billing saves a meaningful amount — 10% off a $475/mo plan is $570/year back in your pocket.

---

**Tips for Building a Reliable Zoopla Scraper**

Beyond the API layer, a few practices make your Zoopla scraper significantly more robust:

**Use UK proxies.** For a UK site like Zoopla, routing through UK IPs is more natural and less likely to trigger geo-based flags. ScraperAPI handles this automatically with the `country_code=gb` parameter.

**Scrape individual property pages without JS rendering when possible.** The hidden Next.js JSON data is available in the static HTML on property detail pages. Save your rendering credits for search pages where you actually need them.

**Check CSS class names after Zoopla updates.** Zoopla deploys frontend changes periodically. If your selectors suddenly break, inspect a fresh page in your browser and update accordingly.

**Use sitemaps for bulk discovery.** For collecting all listings in a category, parsing Zoopla's gzip-compressed sitemap files is far more efficient than paginating through search results.

**Rate limiting isn't necessary when using a scraping API.** ScraperAPI handles retries and distributes requests across its proxy pool — you don't need to add artificial delays on your end, though very aggressive concurrency (hundreds of parallel requests at once) is worth monitoring in the dashboard.

---

**Free Trial and Getting Started**

ScraperAPI doesn't require a credit card to start — new accounts get 1,000 free credits immediately on signup, and a 7-day trial unlocks 5,000 credits for realistic testing against your actual Zoopla targets.

That trial period is genuinely useful. Test it against the specific pages you plan to scrape at scale — a few search pages, a few property detail pages — and check your credit consumption in the dashboard. That gives you real numbers to size your plan against before committing to a monthly subscription.

If you decide the service isn't right for you within the first 7 days, there's a no-questions-asked refund policy.

Annual billing automatically applies a 10% discount if you're ready to commit — no coupon code needed.

👉 [Start your free ScraperAPI trial and test it on Zoopla today](https://www.scraperapi.com/?fp_ref=coupons)

---

**Frequently Asked Questions**

**Is it legal to scrape Zoopla?**

Zoopla's data is publicly available, and scraping publicly accessible pages at respectful rates generally falls within accepted practice. That said, be mindful of GDPR when storing personal data (like agent phone numbers or names of individuals), and don't repurpose entire Zoopla datasets commercially without reviewing the terms of service. If in doubt, consult a lawyer — not a blog post.

**Do I need JavaScript rendering for all Zoopla pages?**

No. Individual property detail pages embed their data in a Next.js JSON object that's accessible in static HTML — no rendering required. Search results pages (the listing index pages) do require JavaScript rendering because listings are loaded dynamically. Use rendering selectively to manage your credit costs.

**What happens if Zoopla changes its HTML structure?**

CSS class names and DOM structure on Zoopla can change after frontend updates. Your scraping API layer (ScraperAPI) handles the anti-bot side; the parsing logic you write is your responsibility to maintain. Checking your selectors after site updates is routine maintenance for any web scraper.

**Can ScraperAPI handle Cloudflare-protected sites?**

Yes, with the `premium=true` parameter. This adds 10 credits to the per-request cost but significantly increases success rates on Cloudflare-protected pages. Zoopla itself doesn't use Cloudflare as of this writing, but many sites in the real estate ecosystem do.

**Does ScraperAPI work for Rightmove too?**

Yes — the same code that works for Zoopla works for Rightmove, OnTheMarket, or any other UK property portal. Just swap the target URL. ScraperAPI's proxy pool and anti-bot handling are site-agnostic.
