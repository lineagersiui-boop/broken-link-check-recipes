# Broken link check recipes: find 404s, redirect chains and noindex headers from your sitemap

Your sitemap is the list of pages you *want* search engines and visitors to reach. Over time some of those URLs start returning 404, move behind a redirect chain, or pick up an `X-Robots-Tag: noindex` header after a deploy. Nobody notices until traffic drops.

This repository has ready-to-use **inputs, automation templates and API recipes** for checking every URL in your sitemap on a schedule, or after each deploy, with [Sitemap URL Health Checker](https://apify.com/lineagersiui/sitemap-url-health-checker), a pay-per-event Apify Actor.

**Scope:** it checks the URLs listed in your sitemap. It doesn't crawl your pages or follow the links inside them, so it won't find a broken link that only appears in page content.

## What gets flagged
Each sitemap URL gets one dataset item with a list of `flags`:

| Flag | Meaning |
|---|---|
| `not_found` | The URL answered 404 or 410 |
| `client_error` | Another 4xx status |
| `server_error` | A 5xx status |
| `redirected` | The URL redirects. `status` is then the final URL's status, or the 3xx itself if the checker stopped following |
| `redirect_chain` | More than one redirect before the final URL |
| `too_many_redirects` | More than 5 redirects; the checker stops following |
| `https_downgrade` | An `https` URL ends up on `http` |
| `noindex_header` | The final response sends an `X-Robots-Tag` header containing `noindex` |
| `redirected_offsite` | A redirect leaves your site; the target is reported, not requested |
| `redirect_blocked_by_robots` | A redirect target is disallowed by robots.txt; reported, not requested |
| `redirect_*` (other) | The redirect target is invalid or not allowed (`redirect_invalid_url`, `redirect_scheme_not_allowed`, `redirect_port_not_allowed`, `redirect_blocked_private_address`); reported, not requested |
| `blocked_by_robots` | robots.txt disallows the URL for this checker, so it was skipped |
| `timeout`, `network_error`, `blocked_private_address` | The URL couldn't be checked |
| `ok` | 2xx with no redirect |

A URL can carry several flags, for example `["not_found", "redirected"]` for a redirect that ends on a 404.

**What it doesn't check:** the `noindex` check reads the HTTP header only. Page bodies are never downloaded, so a `<meta name="robots" content="noindex">` tag in the HTML isn't detected.

Each item has `url`, `status`, `flags`, `redirects`, `redirectStatuses`, `finalUrl`, `lastmod` and `sitemap` (the sitemap file the URL came from). A `SUMMARY` record in the run's key-value store holds the counts: `urlsFound`, `checked`, `billed`, `problems`, `blockedByRobots`, `errors`, sitemaps fetched and failed, and skipped cross-site or invalid URLs.

## Sample output
Real output of the Actor's free demo (empty input, 20 URLs from apify.com sitemaps, not billed, run on 2026-10-27): [`examples/sample-output.json`](examples/sample-output.json). In that run 19 URLs were `ok` and one redirected. Two of the real items:

```json
[
  {
    "url": "https://apify.com/",
    "status": 200,
    "finalUrl": "https://apify.com/",
    "redirects": 0,
    "redirectStatuses": [],
    "flags": ["ok"],
    "lastmod": null,
    "sitemap": "https://apify.com/sitemap/pages.xml",
    "demo": true
  },
  {
    "url": "https://apify.com/alternatives/gmapsextractor-alternative",
    "status": 200,
    "finalUrl": "https://apify.com/alternatives/outscraper-alternative",
    "redirects": 1,
    "redirectStatuses": [308],
    "flags": ["redirected"],
    "lastmod": null,
    "sitemap": "https://apify.com/sitemap/pages.xml",
    "demo": true
  }
]
```

The demo run had no broken URLs. **Illustrative only, not real output:** a URL that redirects twice and ends on a 404 would have this shape, based on the flags documented above:

```json
{
  "url": "https://www.example.com/old-page",
  "status": 404,
  "finalUrl": "https://www.example.com/gone",
  "redirects": 2,
  "redirectStatuses": [301, 302],
  "flags": ["not_found", "redirected", "redirect_chain"],
  "lastmod": "2025-01-15",
  "sitemap": "https://www.example.com/sitemap.xml"
}
```

## Inputs
- [`examples/input-site.json`](examples/input-site.json): one site; the sitemap is found from robots.txt `Sitemap:` lines, or `/sitemap.xml`.
- [`examples/input-sitemaps-problems-only.json`](examples/input-sitemaps-problems-only.json): direct sitemap URLs, a higher URL cap, and problems-only output.

Input fields: `siteUrls` (up to 10), `sitemapUrls` (up to 10; `.xml`, `.xml.gz`, sitemap indexes or plain-text lists), `confirmPermission`, `maxUrls` (default 500, at most 5,000), `onlyProblems`, `timeLimitSecs` (default 1,500, 60 to 3,600).

**Only check sites you own or may check.** The Actor refuses any non-demo run without `"confirmPermission": true`; nothing is checked or charged. It isn't a load-testing tool. The examples use the placeholder `https://www.example.com`: replace it with your own site.

## Run it automatically
1. **Apify Task and Schedule (no code).**
   - In Apify Console, open the Actor, enter your input and save it as a **Task**.
   - Add a **Schedule**, for example weekly, to catch new broken URLs.
   - Open the dataset's *URL checks* view. The `SUMMARY` record has the totals.
2. **GitHub Actions (scheduled, or after a deploy).**
   - Copy [`examples/github-actions/link-check.yml`](examples/github-actions/link-check.yml) into `.github/workflows/` of your own repository.
   - Add a repository secret `APIFY_TOKEN` and a `link-check-input.json` file in the format of the example inputs.
   - The workflow starts a run, waits for it, and **fails the job** when any URL is flagged `not_found`, `client_error`, `server_error` or `too_many_redirects`. It also fails when the run fails (for example without `confirmPermission`) or when no URL got an answer, so a broken setup can't pass as green.
   - It uses the asynchronous API (start, then poll), because the synchronous endpoint waits only about 5 minutes. The full results are saved as a workflow artifact.
   - If the job is cancelled or times out, a final step aborts the Apify run, so it doesn't keep checking and billing.
   - To run it after deploys, enable the `workflow_run` trigger in the file. GitHub disables scheduled workflows after 60 days without repository activity.
3. **API or curl (any scheduler).** See [`examples/run-via-api.md`](examples/run-via-api.md).

**Token and visibility:**
- Use a dedicated Apify API token for automation rather than your main one. If your Apify account lets you scope tokens, limit it to running this Actor and reading its runs' datasets and key-value stores.
- Workflow logs and artifacts of a public repository are visible to anyone. If your list of URLs is sensitive, run the workflow in a private repository.
- Optionally pin the actions (`actions/checkout`, `actions/setup-node`, `actions/upload-artifact`) to commit SHAs.

## How it treats robots.txt
- **Read first:** robots.txt is fetched once per origin before anything else on that origin.
- **Which rules apply:** a `User-agent: SitemapURLHealthChecker` group is used if present, otherwise the `*` group. The longest matching rule wins, `Allow` wins ties, and `*` and `$` are supported.
- **Disallowed URLs** are skipped and flagged `blocked_by_robots`, and aren't billed. Redirect targets are checked against robots.txt too.
- **`Crawl-delay`** is honoured, up to 60 seconds. Requests to one site are never closer than 500 ms apart, one at a time.
- **If robots.txt is missing** (a 4xx answer other than 429), there are no restrictions. If it answers 5xx or 429, times out or fails, everything on that origin is treated as disallowed.
- **Sitemap discovery:** for `siteUrls`, sitemaps come from robots.txt `Sitemap:` lines, otherwise `/sitemap.xml`.
- **Opting out:** a site can block it with `User-agent: SitemapURLHealthChecker` and `Disallow: /`.

## Pricing (set on Apify, may change; check the Actor's Pricing tab)
- **Pay per URL checked:** each URL that returns an HTTP status is one `url-checked` event, **$0.0005** at the time of writing ($0.50 per 1,000 URLs).
- **Example:** checking a 1,000-URL sitemap once a week is about 4,300 URL checks a month, roughly $2.15.
- **Not billed:** the demo, URLs skipped by robots.txt, URLs that time out or fail, and URLs not reached before the time limit.
- **Cap:** you can set a maximum charge per run in Apify's run options. `maxUrls` also caps each run.

## Limits
- Per run: 5,000 URLs, 50 sitemap files nested to a depth of 3, 50 MB per sitemap, 15 s timeout per request, and the time limit you set.
- Only URLs on the same site as their sitemap (the same host ignoring `www.`, its subdomains or its parent domain) are checked. Others are counted in `SUMMARY.skippedCrossSite`.
- Speed: one request at a time per site, at least 500 ms apart, so large sitemaps take a while. 5,000 URLs on one site take at least 40 minutes; raise `timeLimitSecs` for those.
- HEAD requests are used; GET (headers only) is tried when a server refuses HEAD with 403, 405 or 501.
- Results show what the server returned to this checker at run time. Some sites answer automated requests differently from browsers.

## About
- Made by an independent developer. It isn't affiliated with or endorsed by Apify.
- The Actor and this repository were built with the help of AI tools, under the developer's direction. The examples were checked against the Actor's code.
- Issues and suggestions are welcome in this repository's Issues.

License: MIT (examples and templates in this repository).
