---
name: searching-with-serply
description: Fetches live Google Search, Bing Search, Google News, Google Maps, Google Images, Google Jobs, Google Scholar, Amazon product, and Google Video results, or scrapes any URL to markdown or HTML, through the Serply REST API and its hosted MCP server. Use when a task needs current web data, SERP results, or page content beyond the model's training cutoff, or when the user mentions Serply, a SERP API, or scraping a URL.
---

# Searching with Serply

Serply is a REST API (and hosted MCP server) that returns search-engine and
page-scrape results as structured JSON, not raw HTML. Two ways to call it;
pick MCP if it is already configured, otherwise use the REST API directly.

## Auth

Every REST call needs an `X-Api-Key` header. Get a key (2,500 free credits,
no card) at https://app.serply.io/users/sign_up. Never hardcode a real key in
code you write for the user - use `YOUR_API_KEY` as a placeholder and tell
them to set it as an environment variable.

```bash
curl --header 'X-Api-Key: YOUR_API_KEY' 'https://api.serply.io/v1/search?q=query'
```

1 credit = 1 successful, uncached request. Cached responses (most endpoints
cache for several minutes) cost nothing, so retries on the same query during
testing are free.

## REST endpoints

Base URL: `https://api.serply.io`. Most endpoints take the query in the path
(`/v1/search/q=coffee+shops&num=20`); Maps is the one exception (see below).

| Endpoint | Path | Returns |
|---|---|---|
| Google Search | `/v1/search` | `results[]`, `total`, `answer` |
| Google News | `/v1/news` | `feed.entries[]`, `entities[]` |
| Google Maps | `/v1/maps/search/{query}` | `places[]` with lat/long, rating |
| Google Images | `/v1/images` | `results[]` with thumbnail, width/height |
| Google Jobs | `/v1/job/search` | `jobs[]` with employer, is_remote |
| Google Scholar | `/v1/scholar` | `results[]`, `total` |
| Amazon Product | `/v1/product/search` | products with price, rating_stars, prime |
| Google Video | `/v1/video` | `results[]`, `total`, `answer` |
| Bing Search | `/v1/b/search` | `results[]`, `ads[]`, `shoppingAds[]` |
| Page Fetch | `/v1/request` | `content`, `url`, `response_type` (html or markdown) |

Full parameter and response-field reference for each endpoint, including
optional headers like `X-Proxy-Location` (geo-target) and `X-User-Agent`
(desktop/mobile), is at https://serply.io/docs.

**Gotcha:** Google Maps takes query params after a `?`, not packed into the
path like every other endpoint, and does not support `X-Proxy-Location` or
`X-User-Agent` - use `hl`/`gl` for locale instead:

```bash
curl --header 'X-Api-Key: YOUR_API_KEY' \
  'https://api.serply.io/v1/maps/search/coffee%20shops%20in%20Chicago%2C%20IL?num=20&hl=en&gl=us'
```

## MCP server

For an agent that already speaks MCP, point it at the hosted server instead
of writing HTTP calls by hand:

```bash
claude mcp add --transport http serply https://api.serply.io/mcp \
  --header "X-Api-Key: YOUR_API_KEY"
```

For clients without native remote-MCP support, bridge with `mcp-remote` in
the client's MCP config, setting the same header.

Available tools (call as `serply:tool_name`):

- `serply:google_search` - query, num, start, proxy_location, device
- `serply:bing_search` - query, proxy_location, device
- `serply:google_news_search` - query, ceid, proxy_location, device
- `serply:scrape_url` - url, response_type
- `serply:google_scholar_search` - query, num, proxy_location, device
- `serply:google_video_search` - query, num, proxy_location, device
- `serply:google_jobs_search` - query, proxy_location, device
- `serply:amazon_product_search` - query, proxy_location, device

There is no Maps tool over MCP yet - use the REST endpoint above for Maps.

## Errors

`429` means the rate limit was hit; back off and retry. `502` on Maps means
the upstream parse failed - retry once before reporting a failure. Full
error-response shape: https://serply.io/docs/guides/errors.
