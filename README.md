# How to Use ScraperAPI: The Smallest Working Request, When MCP Beats the SDK, and How Many Pages Your Credits Actually Cover

ScraperAPI's smallest usable call is one GET request: send `api_key` and `url` to `https://api.scraperapi.com`, and the response body is the HTML of the page you asked for. Proxy rotation, CAPTCHA handling and JavaScript rendering happen on their side, which is the whole reason to pay for it instead of running your own proxy pool.

Before you write any integration code, prove the key works. A wrong or inactive key returns `401`, and retrying with that same key just wastes time.

> **If you want it running tonight:** copy the key from the dashboard, export it as an environment variable, and run the `httpbin.org/ip` check below. Stop once you see an `origin` IP in the response. Choose MCP, SDK or plain REST only after that works.
>
> **On cost:** the free plan gives 1,000 API credits with 5 concurrent connections. Paid plans run from **$49/month** (Hobby, 100,000 credits, 20 threads) up to **$1,975/month** (Advanced, 21.5M credits, 500 threads), and annual billing takes 10% off.
>
> 👉 [Compare every ScraperAPI plan and current price](https://www.scraperapi.com/pricing?fp_ref=coupons)

## ScraperAPI plans and prices

All paid plans include JS rendering, premium residential and mobile IPs, anti-bot bypass, autoparse, rotating proxies, CAPTCHA handling, desktop and mobile user agents, automatic retries and unlimited bandwidth. What changes between tiers is the credit allowance, how many requests can run at once, and where you can geotarget.

| Plan | API credits / month | Concurrent threads | Monthly | Annual (per month) | Start |
| --- | --- | --- | --- | --- | --- |
| Free | 1,000 | 5 | $0 | — | [Start free](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | 100,000 | 20 | $49 | $44.10 | [Get Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 1,000,000 | 50 | $149 | $134.10 | [Get Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | 3,000,000 | 100 | $299 | $269.10 | [Get Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 | $475 | $427.50 | [Get Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | 10,500,000 | 300 | $975 | $877.50 | [Get Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | 21,500,000 | 500 | $1,975 | $1,777.50 | [Get Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 500+ | Custom | Custom | [Request Enterprise pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |

A few differences are easy to miss in the table. Hobby and Startup can only geotarget the US and EU; country-level geotargeting anywhere in the world starts at Business. Analytics history is capped at 30 days on Hobby and Startup and unlimited from Business up. Pay-as-you-go overage, where you keep scraping past your limit at a fixed per-credit rate up to a cap you set, is available on Scaling, Professional, Advanced and Enterprise. Hobby, Startup and Business instead get a `429` when credits run out, with auto-upgrade available if you opt in. Credits reset on each monthly renewal date and unused ones are not carried forward.

## What a credit actually buys

A standard page costs 1 credit. That number is the one everyone quotes and the one that misleads most tutorials, because the multiplier is set per domain, not per plan.

| Target | Credits per request |
| --- | --- |
| Standard page | 1 |
| Amazon | 5 |
| Google or Bing (including subdomains) | 25 |
| LinkedIn | 30 |
| Sites behind Cloudflare, DataDome or PerimeterX | +10 when bypass is triggered |

JS rendering and premium proxies push the multiplier up further. That changes the real capacity of each plan by an order of magnitude:

| Plan | Standard pages | Amazon products | Google searches |
| --- | --- | --- | --- |
| Hobby ($49) | 100,000 | 20,000 | 4,000 |
| Startup ($149) | 1,000,000 | 200,000 | 40,000 |
| Business ($299) | 3,000,000 | 600,000 | 120,000 |

If your project is 90% plain HTML pages, Hobby's 100,000 credits is a lot of headroom. If it's Google SERPs, the same plan covers 4,000 searches, so a daily keyword sweep across 200 terms blows through a month in 20 days. The dashboard's Domain Cost Estimator gives the exact cost for a specific URL, and you can set `max_cost` per scrape so a single expensive request can't quietly eat your budget.

Watch the plan boundaries too. Startup covers 1M credits and Business covers 3M, but there is no overage tier below Scaling, so a job that needs 1.1M credits costs $299 instead of $149.

👉 [Check the plan whose thread count matches your workload](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Step 1: Get the key and prove it works

Copy the key from the dashboard, then put it in an environment variable. Hardcoding it in code or a config file is how keys end up in commit history.

bash
# macOS / Linux
export SCRAPERAPI_API_KEY=<your-key>

# Windows PowerShell
$env:SCRAPERAPI_API_KEY = "<your-key>"


Then run one real request against a target that can't fail for content reasons:

bash
curl "https://api.scraperapi.com/?api_key=$SCRAPERAPI_API_KEY&url=https://httpbin.org/ip"


A JSON body containing an `origin` IP means the key is live. A `401` means the key is wrong or inactive, and the correct response is to surface the error rather than retry. The same check in Python:

python
import os, requests

r = requests.get(
    "https://api.scraperapi.com/",
    params={"api_key": os.environ["SCRAPERAPI_API_KEY"], "url": "https://httpbin.org/ip"}
)
print(r.status_code, r.text[:200])


A `200` with valid JSON means auth, credits and parameters are all wired correctly. Doing this before you build the integration catches problems while they're still one line of output instead of a stack trace inside your app.

## Step 2: Decide what's calling ScraperAPI

The path you pick depends on what consumes the data, not on which one sounds more advanced.

| What you're doing | Where to start |
| --- | --- |
| An LLM agent needs a page or a SERP during this session | MCP server |
| ScraperAPI inside app code | SDK or REST |
| A handful of requests, or no Python/Node on the machine | Plain REST |
| A batch of 100+ URLs | Async Jobs API |
| Recurring jobs you don't want to babysit | DataPipeline |

**MCP server, if the consumer is a model.** The remote server is the one to use: no install, and it exposes up to 22 tools covering scrape, Google, Amazon, Walmart, eBay, Redfin and crawler. Drop this into `claude_desktop_config.json`, your Claude Code project settings, or any MCP client config:

json
{
  "mcpServers": {
    "ScraperAPI": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://mcp.scraperapi.com/mcp",
        "--header",
        "Authorization: Bearer ${SCRAPERAPI_API_KEY}"
      ]
    }
  }
}


`npx mcp-remote` handles the remote connection, so there's no separate install step, but `SCRAPERAPI_API_KEY` has to be set in the environment the client runs in.

The self-hosted server is a different product in practice. `pip install scraperapi-mcp-server` (Python 3.11+) and you configure it with `"command": "python"`, `"args": ["-m", "scraperapi_mcp_server"]` and `"env": { "API_KEY": "<YOUR_SCRAPERAPI_API_KEY>" }`. It exposes **only the `scrape` tool**. If you want structured Google results from the local variant, you call `scrape` on a Google search URL with `autoparse: true` and `outputFormat: "json"`. People install the local server, see a single tool where the documentation promised twenty-two, and assume something broke. Nothing broke.

One thing worth knowing before you point an agent at it: queries, URLs and page content get transmitted to ScraperAPI, so don't route anything through it that you couldn't send to a third party.

**SDK, if you're writing application code.** `pip install scraperapi-sdk` gives you `ScraperAPIClient(os.environ["SCRAPERAPI_API_KEY"]).get("https://example.com", render=True, country_code="us")`. On Node it's `npm install scraperapi-sdk`, then `require('scraperapi-sdk')(process.env.SCRAPERAPI_API_KEY)` and `scraperapi.get('https://example.com', { render: true, country_code: 'us' })`. PHP, Ruby and Java SDKs exist too.

**Plain REST, if it's five requests.** Base URL is `https://api.scraperapi.com/`, auth is the `api_key` query parameter, and async jobs take `apiKey` in the JSON body instead. The four calls that cover most early work:

bash
# basic page
curl "https://api.scraperapi.com/?api_key=$SCRAPERAPI_API_KEY&url=https://example.com"

# page that needs a browser
curl "https://api.scraperapi.com/?api_key=$SCRAPERAPI_API_KEY&url=https://example.com&render=true"

# Google SERP as JSON
curl "https://api.scraperapi.com/structured/google/search?api_key=$SCRAPERAPI_API_KEY&query=web+scraping"

# Amazon product as JSON
curl "https://api.scraperapi.com/structured/amazon/product?api_key=$SCRAPERAPI_API_KEY&asin=B09V3KXJPB"


For batch work, POST to `https://async.scraperapi.com/jobs` with `{"apiKey": "...", "url": "..."}` and poll `https://async.scraperapi.com/jobs/<jobId>?apiKey=...` until it finishes.

If you're not sure which API fits the job, the mapping is short: any URL as HTML or markdown goes to the Standard API; React, Vue and SPA pages need the Standard API plus `render=true`; Amazon, Google, Walmart, eBay and Redfin data belongs on the Structured Data API; 100+ URLs go to Async Jobs; scheduled runs go to DataPipeline.

## Step 3: The parameters that cancel each other out

These bite after the first successful request, when you start tuning. Several combinations are not just discouraged but silently dropped.

| Combination | What happens |
| --- | --- |
| `premium` + `ultraPremium` | Mutually exclusive, pick one |
| `sessionNumber` + `premium`/`ultraPremium` | Not compatible |
| `retry404` + `premium`/`ultraPremium` | Not available |
| `keepHeaders` + `ultraPremium` | Not applied |
| `customHeaders` + `ultraPremium` | Headers discarded |
| `desktopDevice` + `mobileDevice` | One or the other |
| `outputFormat` `csv`/`json` without `autoparse` | Only works on autoparse-supported sites; otherwise you get HTML |
| `waitForSelector` or `instructionSet` | Implicitly turns on `render` |
| `zipCode` | Amazon US only |

Two more behaviours worth knowing. Adding any `customHeaders` entry switches `keepHeaders` on automatically, and overriding ScraperAPI's own header management tends to lower success rates, so use it only when a site genuinely requires a specific header. And `followRedirect` defaults to `true`, which is what you want until you're debugging a redirect chain.

REST parameter names use snake_case (`ultra_premium`), while SDK and node fields use camelCase (`ultraPremium`). Mixing them produces a request that succeeds and quietly ignores your setting.

## Step 4: What to do with each status code

Most "how to use" guides stop before this part, which is why the first `403` sends people to support.

| Code | Meaning | Action |
| --- | --- | --- |
| `200` | Success | Parse the body |
| `401` | Invalid API key | Surface the error, don't retry |
| `403` | Blocked by the target site | Retry with `premium=true`, `ultra_premium=true`, or a different `country_code` |
| `429` | Concurrent limit hit | Back off; move batch work to async |
| `500` / `503` | Transient failure | Retry with exponential backoff, 3 to 5 attempts |

The `429` deserves a note on billing: going over your limit doesn't incur a charge, the request is refused rather than queued. That distinction matters when you're deciding whether to add retry logic or upgrade the thread count.

👉 [See which plan raises your concurrent thread limit](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Step 5: Get the same data for fewer credits

For any platform with a dedicated structured endpoint, use it instead of scraping HTML and parsing it yourself. ScraperAPI's own guidance is that structured endpoints return clean JSON with no parsing logic, are more reliable, and often cost fewer credits than raw scraping with `render=true`. That last part is the one people ignore: a rendered Amazon page can cost more than the 5-credit structured product endpoint that returns the fields you actually wanted.

Structured endpoints exist for Amazon (product, search, offers), Google (search, jobs, news, shopping, maps), eBay (search, product), Walmart (search, category, product, review) and Redfin (for sale, for rent, search, agent). They all follow `https://api.scraperapi.com/structured/{platform}/{endpoint}`, return JSON by default, and most accept `csv`.

For volume, the Async Jobs API takes the concurrency problem off your plate. For multi-page extraction, the Crawler follows links from a seed URL, and it has four required fields rather than one: `startUrl`, either `maxDepth` or `crawlBudget` (a credit ceiling), `urlRegexpInclude` (use `.*` to allow the whole site), and `callbackUrl`, which receives both successful and failed scrapes plus a job summary when the run ends. `urlRegexpExclude` is worth setting on e-commerce sites so you don't crawl every product variant. `scheduleInterval` takes `once`, `hourly`, `daily`, `weekly` or `monthly`.

Two hard limits apply across all of this. Requests are capped at **50MB**, which matters if you're pulling PDFs or large files rather than pages. And the account-level concurrency limit is what produces `429`s, not the size of the URL list.

## Using it inside Claude, n8n, or your own agent

If you already live in a workflow tool, the official n8n node covers four resources: a single-URL API request, structured data endpoints, the crawler, and AI Parser operations. Install it by searching "ScraperAPI" in the node panel, paste the dashboard key into the credential, and the credential tests connectivity for you. Responses come back shaped as `{"resource": "api", "response": {"body": ..., "headers": ..., "statusCode": 200, "statusMessage": "OK"}}`, so downstream nodes read from `response.body`.

The AI Parser is the piece to treat with some caution. The official node exposes the full set of operations (create, get, list, parse a URL, update, delete), you feed it one to three example URLs with the same layout, and it generates a parser asynchronously with status `GENERATING`, `FINISHED` or `FAILED`. Version `-1` always points at the latest. Adding or changing fields triggers a new version, while renaming and removing fields apply immediately. Documentation elsewhere still labels it a closed beta, so don't build a production pipeline that depends on its output shape staying fixed.

For LangChain, there's an official integration that lets an agent fetch pages or pull Google and Amazon results without you writing the request layer at all.

## Which setup fits you

Match the setup to the job and skip everything else:

- **One page, once.** Free plan, one curl. No SDK, no MCP server.
- **A script that fetches a few hundred pages a month.** Hobby at $49 covers 100,000 standard credits. Add `render=true` only on the pages that return empty content without it.
- **Amazon, Google, Walmart, eBay or Redfin data at any volume.** Budget at 5 or 25 credits per request, not 1, and use the structured endpoints rather than parsing HTML.
- **100+ URLs per run.** Async Jobs, with `max_cost` set so one bad domain can't drain the account.
- **Recurring scrapes.** DataPipeline, or the Crawler with a schedule interval and a `crawlBudget` ceiling.
- **An agent that needs web data mid-conversation.** The remote MCP server, not the local one, unless you're genuinely fine with a single `scrape` tool.

Three things to confirm before you pay: what multiplier your actual targets carry (the Domain Cost Estimator settles this in a minute), whether your monthly volume sits near a plan boundary where the next tier costs $150 more, and that unused credits expire at renewal rather than rolling over. Nothing about the pricing punishes starting small, so run the free credits down on real targets first and let the multiplier data pick the plan.

👉 [Start on the free plan, then move up when a real job outgrows it](https://www.scraperapi.com/pricing?fp_ref=coupons)
