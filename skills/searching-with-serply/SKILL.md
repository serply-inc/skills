---
name: searching-with-serply
description: Fetches live Google Search, Bing Search, Google News, Google Maps, Google Images, Google Jobs, Google Scholar, Google Video, Amazon product, eBay, and Reddit results (subreddit listings, user history, posts, and comment threads), or scrapes any URL to markdown or HTML, through the Serply REST API and its hosted MCP server. Use when a task needs current web data, SERP results, Reddit discussion, or page content beyond the model's training cutoff, or when the user mentions Serply, a SERP API, or scraping a URL.
license: MIT
metadata:
  author: serply-inc
  version: "1.1"
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
curl --header 'X-Api-Key: YOUR_API_KEY' 'https://api.serply.io/v1/search?q=query&num=10'
```

Send the query as an ordinary query string (`?q=...&num=...`). Older examples
and some client libraries pack the same parameters into the path
(`/v1/search/q=query&num=10`); the API accepts both and returns the same
results, but the path form is fragile: a literal `%` in the query gets a
`400 Bad Request` from the edge, and a literal `?` or `#` silently truncates
the query, so you get results for a shorter search and `num` falls back to
its default. Percent-encode the query and use `?q=`.

1 credit = 1 successful, uncached request. Cached responses (most endpoints
cache for several minutes) cost nothing, so retries on the same query during
testing are free.

## REST endpoints

Base URL: `https://api.serply.io`. Every search endpoint takes `q` and its
options as a query string (`/v1/search?q=coffee+shops&num=20`). Maps takes
the query as a path segment and Reddit takes ids and names as path segments;
both take their options as query params (see below).

| Endpoint | Path | Returns |
|---|---|---|
| Google Search | `/v1/search` | `results[]`, `answers[]`, `related_searches` |
| Google News | `/v1/news` | `feed.entries[]`, `entities[]`; add `&resolve_links=true` for publisher URLs instead of `news.google.com` redirects |
| Google Maps | `/v1/maps/search/{query}` | `places[]` with lat/long, rating |
| Google Images | `/v1/image` | `image_results[]` with `image.src`, `link`, `original_image` |
| Google Jobs | `/v1/job/search` | `jobs[]` with employer, is_remote |
| Google Scholar | `/v1/scholar` | `articles[]` with author, citations count |
| Amazon Product | `/v1/product/search` | products with price, rating_stars, prime |
| Google Video | `/v1/video` | `results[]`, `total`, `answer` |
| Bing Search | `/v1/b/search` | `results[]`, `ads[]`, `shoppingAds[]` |
| eBay Search | `/v1/ebay/search` | `results[]` with title, link, price |
| Reddit | `/v1/reddit/...` | Reddit's own listing JSON - see below |
| Page Fetch | `/v1/request` | the page body as plain text - **POST only**, see below |

Full parameter and response-field reference for each endpoint, including
optional headers like `X-Proxy-Location` (geo-target) and `X-User-Agent`
(desktop/mobile), is at https://serply.io/docs.

**Gotcha:** Google Maps takes the query itself as a path segment
(`/v1/maps/search/{query}`), not as `q=`, and does not support
`X-Proxy-Location` or `X-User-Agent` - use `hl`/`gl` for locale instead:

```bash
curl --header 'X-Api-Key: YOUR_API_KEY' \
  'https://api.serply.io/v1/maps/search/coffee%20shops%20in%20Chicago%2C%20IL?num=20&hl=en&gl=us'
```

**Gotcha:** Reddit has no `q` at all - the subreddit, user, or post id is a
path segment and the options are query params. Five endpoints:

```bash
curl --header 'X-Api-Key: YOUR_API_KEY' \
  'https://api.serply.io/v1/reddit/subreddit/python?limit=25&sort=hot&t=all'
```

| Path | Returns |
|---|---|
| `/v1/reddit/subreddit/{sub}` | Reddit's listing: `data.children[]` of `kind: t3` posts |
| `/v1/reddit/subreddit/{sub}/about` | one object describing the subreddit |
| `/v1/reddit/user/{username}` | that user's posts, same listing shape |
| `/v1/reddit/post/{id}` | one post object - body at `selftext` |
| `/v1/reddit/comments/{id}` | `{cached, data: [post listing, comment listing]}` |

Reddit's own JSON passes through untouched, so `comments/{id}` is the one that
is not a bare listing: the thread is at `data[1].data.children`, each child a
`kind: t1` comment whose `replies` is either a listing object or `""` (not
`null`) when there are none. `post/{id}` does that unwrapping for you and is
the right call when you want the post body rather than the discussion; add
`?with_comments=true` to get both. An id that does not exist returns `404`,
not `502`. `sort` accepts Reddit's values (`hot`, `new`, `top`, `confidence`)
and `limit` is 1-100, default 25.

**Gotcha:** Page Fetch is the one endpoint you `POST` to, with a JSON body.
`GET https://api.serply.io/v1/request?url=...` returns `405 Method Not
Allowed`. There is a legacy path-packed `GET /v1/request/url=<urlencoded>`
that does respond, but it ignores `response_type` and only ever hands back
raw HTML. Use `POST`.
It also returns the page body directly as text - not a JSON object, so do not
try to parse it or reach for a `content` key:

```bash
curl --request POST \
  --header 'X-Api-Key: YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{"url":"https://example.com","response_type":"markdown"}' \
  'https://api.serply.io/v1/request'
```

`response_type` takes `markdown` or `full`; omitting it gives you the raw HTML
as plain text. Only `full` returns JSON - an object carrying the HTML in `data`
alongside `status` and `headers`. `markdown` and the default are
both plain text bodies, so `json.loads` on them throws.

Pass `markdown` for anything headed into a model context - it strips the nav,
scripts, and boilerplate that would otherwise dominate the tokens. Note that
markdown output escapes literal periods in decimals (`29\.1`), so strip
backslashes before substring-matching a number in the response.

**Gotcha:** `related_questions` (Google's "People also ask" box) is frequently
returned empty even on question-shaped queries. What usually populates instead
is `related_searches.text[]`, whose entries carry `result_type:
"people_also_search"` - Google's "People also search for". They are different
boxes; do not treat one as the other when doing keyword research.

## MCP server

For an agent that already speaks MCP, point it at the hosted server instead
of writing HTTP calls by hand:

```bash
claude mcp add --transport http serply https://api.serply.io/mcp \
  --header "X-Api-Key: YOUR_API_KEY"
```

For clients without native remote-MCP support, bridge with `mcp-remote` in
the client's MCP config, setting the same header.

Fourteen tools, called as `serply:tool_name`:

- `serply:google_search` - query, num, start, proxy_location, device
- `serply:bing_search` - query, proxy_location, device
- `serply:google_news_search` - query, ceid, proxy_location, device
- `serply:google_video_search` - query, num, proxy_location, device
- `serply:google_scholar_search` - query, num, proxy_location, device
- `serply:google_jobs_search` - query, proxy_location, device
- `serply:google_maps_search` - query, num, hl, gl
- `serply:amazon_product_search` - query, proxy_location, device
- `serply:scrape_url` - url, response_type
- `serply:reddit_subreddit_posts` - subreddit, limit, sort, t, after
- `serply:reddit_subreddit_about` - subreddit
- `serply:reddit_user_posts` - username, limit, sort, t, after
- `serply:reddit_post` - post_id, with_comments, sort, max_depth
- `serply:reddit_post_comments` - post_id, sort, max_depth

Everything except `google_maps_search` returns rendered markdown rather than
JSON - readable as-is, no parsing step. Maps returns a structured object with
a `places[]` array, and is also the one tool that takes `hl`/`gl` instead of
`proxy_location`/`device`.

The Reddit tools take ids and names the way you already have them: `r/python`
or `python`, `u/spez` or `spez`, `t3_1vfemi1` or `1vfemi1`. Prefer
`reddit_post` when you want the post itself (pass `with_comments=true` to get
the thread too) and `reddit_post_comments` when you only want the discussion.
A post id that does not exist comes back as a plain "No post found" answer,
not a tool error - do not retry it.

## Errors

`429` means the rate limit was hit; back off and retry. `502` means the
upstream fetch or parse failed - it is transient and not specific to Maps
(Page Fetch throws it too), so always retry once before reporting a failure.
`405` on Page Fetch means you used `GET` instead of `POST`. `404` means the path
is wrong or the thing is not there, and is never worth retrying: from
`/v1/reddit/post/{id}` it is a real answer that the post does not exist, and
from any endpoint it also means a misspelled path or an empty query segment
(`/v1/image/` with nothing after the slash). Full
error-response shape: https://serply.io/docs/guides/errors.

A `200` is not proof the call was well-formed. A search endpoint given a bare
path segment (`/v1/search/coffee%20shops`) or a path-packed query with an
unencoded `?` or `#` still returns `200`, just for a different query than you
meant. Image is the sharpest case: `/v1/image/coffee&num=5` searches for the
literal text `coffee&num=5` and returns a full set of unrelated pictures with
no error. Use the query string - `/v1/image?q=coffee&num=5` - and, when a
result set looks off, check that the titles match the query before trusting
it. Image also returns 20 results regardless of `num`.
