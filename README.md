# playwright scraping python: when your script works locally but gets blocked in production — how to fix it with ScraperAPI, plus the real cost math

The classic turning point in a Playwright Python project is the day the script moves from your laptop to production and starts getting 403s and CAPTCHAs. Your code didn't get worse. Default headless browsers leak signals that protection systems pick up instantly: the `navigator.webdriver` flag is set, the TLS fingerprint doesn't match real Chrome, real plugins and rendering characteristics are missing, and your datacenter IP is already on a watchlist. Cloudflare, DataDome, and Akamai combine those signals into a score, and Playwright out of the box scores like a bot.

This article covers two things. First, how to write Playwright scrapers that hold up — including the hidden-API trick most tutorials skip. Second, when you need actual anti-blocking capability, how to wire a service like ScraperAPI into Playwright correctly, including one integration pitfall the official docs explicitly warn about, and the credit math that determines what it really costs.

> **Quick take:** most people running standard Playwright scrapes are fine starting on ScraperAPI's Hobby plan ($49/mo, or $44.10 billed annually — 100,000 credits, 20 concurrent threads). Not sure? Use the free 7-day trial to measure real credit burn on your exact targets before paying anything. 👉 [Start the 7-day free trial — 5,000 credits, no card required](https://www.scraperapi.com/signup?fp_ref=coupons)

## When Playwright is enough, and when it needs another layer

If the data sits in the initial HTML, you don't need Playwright at all. Its job is the other case: pages where content is painted in by JavaScript after load. Point `requests` at a marketplace, dashboard, or social feed and you get an almost empty shell. Playwright drives a real browser (Chromium, Firefox, or WebKit), waits for the scripts to run, and hands you the fully rendered DOM.

Setup on the Python side is two steps, and the second one is the one people forget:

1. `pip install playwright` — installs the library only
2. `playwright install chromium` — downloads the browser binary; without it, everything fails on first run

One habit saves most of the debugging time: for asynchronously loaded content, wait for it with `page.wait_for_selector("div.quote")`. Playwright's action methods auto-wait — `page.click()` and `page.inner_text()` wait for an element to be present, visible, and stable before acting, up to a configurable timeout. That's the core advantage over Selenium: most of the flaky timing code that makes Selenium scripts fragile simply isn't needed here.

The classic sandbox for seeing this in action is quotes.toscrape.com/js/ — the quotes are rendered by JavaScript, so a plain `requests.get` returns nothing, while Playwright loads the page, runs the scripts, and only then reads the DOM.

Two techniques most guides skip, both worth learning early:

- **Intercept the hidden JSON API.** Data loaded on scroll usually comes from a background endpoint. Use `page.on("response")` to listen to network traffic, filter for URLs containing `/api/` with status 200, and `response.json()` gives you structured data directly. Once you've found the endpoint, you can often drop the browser entirely and call it over plain HTTP. As one 2026 tutorial puts it, "The best browser scrape is frequently the one you replace with a direct API call."
- **Reuse login state.** `page.fill` and `page.click` to submit the form, `wait_for_url` to catch the redirect, then `page.context.storage_state(path="auth.json")` to save the session. Next run, `browser.new_context(storage_state="auth.json")` starts already logged in.

The moment your targets sit behind Cloudflare or DataDome, none of the above fixes the fingerprint and IP problem. You then have two paths: keep patching Playwright with anti-detection tricks, which means continuous maintenance every time the protection rules update, or hand the request to infrastructure built for exactly this. That's where ScraperAPI sits: you send it a URL, it routes the request through a rotating pool of 40M+ IPs across 50+ countries, optionally renders JavaScript with headless Chrome, handles CAPTCHAs, and returns the HTML. It's infrastructure, not a platform — no storage, no scheduling, no pre-built scrapers; your code keeps doing all of that.

## Integrating ScraperAPI with Playwright: the one recommended method and one guaranteed pitfall

The officially recommended method is the API endpoint approach: don't put the proxy into your browser config. Wrap the target URL as a ScraperAPI endpoint and have Playwright open that endpoint. The official Playwright guide (updated July 2025) shows the pattern in Node.js; the same structure works identically from Python:

javascript
const scraperApiUrl = `https://api.scraperapi.com?api_key=${SCRAPERAPI_KEY}&url=${encodeURIComponent(targetUrl)}`;
// page.goto(scraperApiUrl, { waitUntil: 'domcontentloaded' })


Optional parameters append to the URL:

- `render=true` — enable JavaScript rendering; turn this on for JS-heavy sites
- `country_code=us` — use a US-based IP
- `session_number=123` — stick to the same proxy session
- `premium=true` — use premium proxies

Store the API key in a `.env` file, and note the guide's explicit reminder: no quotes around the key.

**The pitfall: don't put ScraperAPI's proxy port into Playwright's `launch()` options.** The proxy port (`proxy-server.scraperapi.com:8001`) exists for people who already have a proxy pool and want the API behind a proxy front-end — username `scraperapi`, password your API key. It works fine in ordinary HTTP clients. It fails in Playwright, because Playwright's proxy configuration only accepts Basic Auth or IP-based authentication, while ScraperAPI's port expects the API key as a query parameter. The two authentication models don't mix, and the actual error you'll get is `Proxy Authentication Required`. The official guide is blunt about it: "Playwright doesn't support query string authentication in proxy URLs." So the rule is simple: when using ScraperAPI, avoid Playwright's own proxy settings entirely and go through the API endpoint.

Two details matter if you use proxy mode in other tools (requests, Scrapy, anything that accepts a normal proxy): your code must be configured to not verify SSL certificates for requests to route correctly, and extra features are enabled through the username — `scraperapi.render=true` for JS rendering, multiple parameters joined by periods. If you want to keep SSL verification, you need to download and manually trust ScraperAPI's Proxy CA certificate (via mmc on Windows, Keychain Access on macOS, or installing proxyca.pem on Linux and updating the system CA store).

## Credits are the real cost: the number on the pricing table can't be taken at face value

ScraperAPI bills by credit, and "1 request = 1 credit" only holds for standard pages. The official pricing page states it plainly: a standard page costs 1 credit, Amazon costs 5, Google and Bing (including their subdomains) cost 25, and LinkedIn costs 30. Sites behind Cloudflare, Datadome, or PerimeterX add 10 credits per request when the block is bypassed. These multipliers stack.

That's how "100,000 credits sounds like a lot" and "my credits burned out in days" describe the same person. If your target is search engine result pages at 25 credits each, the Hobby plan's 100,000 credits is really about 4,000 requests. One third-party pricing analysis put the pattern memorably: half the people who sign up watch their credits disappear in three days instead of thirty, because they sized the plan off the headline number instead of their actual domain mix.

Two tools turn that uncertainty into something you can control:

- The **Domain Cost Estimator** in your dashboard: enter any URL and see its exact per-request cost
- The **`max_cost`** request parameter: cap what any single scrape can consume

Also budget differently if you plan to use **DataPipeline**, ScraperAPI's no-code scheduled jobs feature: it burns credits at its own rate, not the standard API rates listed above, so estimates based on the standard table will be wrong.

Run your real target URLs through the free trial before buying anything — that measurement beats any plan recommendation, including the ones in this article.

## Full plan comparison (current official pricing)

Every plan carries the same feature list: JS rendering, premium residential and mobile IPs, JSON auto-parsing, rotating proxy pools, custom headers, CAPTCHA and anti-bot detection, custom sessions, automatic retries, unlimited bandwidth, and a 99.9% uptime guarantee. Annual billing takes 10% off.

| Plan | Monthly (annual) | API credits | Concurrent threads | Geotargeting | Get it |
| --- | --- | --- | --- | --- | --- |
| Free account | $0 | 1,000 | 5 | — | [Sign up for free credits](https://dashboard.scraperapi.com/signup?fp_ref=coupons) |
| Hobby | $49 ($44.10) | 100,000 | 20 | US & EU only | [Get Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | $149 ($134.10) | 1,000,000 | 50 | US & EU only | [Get Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | $299 ($269.10) | 3,000,000 | 100 | Country-level | [Get Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | $475 ($427.50) | 5,000,000 | 200 | Country-level | [Get Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | $975 ($877.50) | 10,500,000 | 300 | Country-level | [Get Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | $1,975 ($1,777.50) | 21,500,000 | 500 | Country-level | [Get Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | Custom quote | 22M+ | 500+ | Country-level | [Contact sales](https://www.scraperapi.com/contact-sales/?fp_ref=coupons) |

Two separate free entries, easy to confuse: the **free account** (1,000 credits, max 5 concurrent connections, via dashboard signup) and the **7-day trial** (5,000 credits, no credit card required). You can stack the trial's 5,000 credits on top of testing needs, and if you need more credits specifically for testing, ScraperAPI says to contact support.

A few rules that only show up in the fine print on the pricing page:

- **Over the concurrency limit:** you get a 429 response and the request is rejected outright — not queued, not billed. Lower your concurrency and retry.
- **When credits run out, scraping pauses by default** until the next cycle. Hobby through Professional can opt into auto-upgrade (the account moves to the next plan automatically when credits exhaust); Scaling and above can use pay-as-you-go, continuing at a fixed per-credit rate up to a monthly cap you set yourself.
- **Annual plans can renew early** for a fresh batch of credits.

## Four decision rules for picking a plan

Standard pages only — blogs, news sites, most HTML targets → **Hobby is enough.** 100,000 credits at the 1x rate means 100,000 requests, and $49/month covers most personal projects.

E-commerce data → **start at Startup.** Amazon-class sites cost 5 credits per request, so Hobby's 100,000 credits is really 20,000 Amazon requests. Startup's 1M credits stretches to 200,000.

Need IPs outside the US and EU → **go straight to Business.** Hobby and Startup are geotargeted to US and EU only, and there's no way to unlock other countries' IPs on those tiers — country-level targeting starts at Business ($299).

Unpredictable workload where a mid-month cutoff would hurt → **start at Scaling.** It's the lowest tier with pay-as-you-go, so scraping continues past your allocation at a fixed rate under a cap you set, instead of halting. 👉 [Compare all plans and annual pricing on the official page](https://www.scraperapi.com/pricing?fp_ref=coupons)

And the honest reverse case: if your target sites have no protection and the data is already in the initial HTML, a plain HTTP request library does the job. Credits spent on ScraperAPI in that scenario pay for problems you never had.

On trust: a third-party pricing analysis notes the documentation scores 4.9/5 for ease of use on Capterra, which the analysis flags as rare in this category — setup is measured in minutes, and the actual difficulty is in understanding what your credits buy, not in using the product.

## FAQ

**Can Playwright bypass Cloudflare on its own?** Out of the box, effectively no. The automation flags, TLS fingerprint, and datacenter IP of a default headless browser are all picked up by modern protection systems. Anti-detection patches exist but need ongoing maintenance whenever the protection rules change. A service like ScraperAPI outsources that maintenance, at the price of paying per credit.

**With ScraperAPI and Playwright together, do I still need my own browser?** With the endpoint method, the browser's only job is to open the wrapped URL — the actual JS rendering and proxying happen on ScraperAPI's side (remember `render=true`). And if your page's data is reachable through a hidden JSON API, you can skip the browser entirely and hit the endpoint with requests instead, which is faster and cheaper on both fronts.

**What can I do with the free tier?** Sign-up gives 1,000 credits with 5 concurrent connections, and the 7-day trial adds 5,000 credits with no credit card. The practical move — one that shows up in every serious breakdown of the service — is to spend the free credits measuring real per-request costs on your actual target URLs before choosing a tier. Need extra testing credits? Support can grant more.

**How do I get warned before credits run out?** You don't, automatically. ScraperAPI has no proactive credit alerts — you check usage in the dashboard yourself or set your own reminder. If you're the forgetful type, pay-as-you-go's monthly spending cap doubles as a safety net.

## Two things to check before you buy

First, spend the trial's 5,000 credits on your real target URLs, note what the Domain Cost Estimator reports per request, multiply by your monthly volume, and add 20–30% headroom — the resulting number maps directly onto a plan tier. Second, confirm whether you genuinely need IPs beyond the US and EU; that single answer moves you from the $49–$149 range to Business and above, and it's the biggest fork in the whole pricing table. Once both are settled, 👉 [pick your plan on ScraperAPI's pricing page](https://www.scraperapi.com/pricing?fp_ref=coupons) — annual billing locks in a flat 10% discount.
