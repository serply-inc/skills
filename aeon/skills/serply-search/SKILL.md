---
name: serply-search
description: Google web and news search through the Serply API, with ranked, cited results and optional country and freshness filters
metadata:
  title: Serply Search
  mode: read-only
  category: research
  var: ""
  tags:
    - web
    - search
    - news
    - research
  requires:
    - SERPLY_API_KEY
---

> **${var}** — Search query. When empty, searches for current notable developments in AI and technology.

Today is ${today}. Search Google results through the [Serply](https://serply.io) API for **${var}** and deliver a short, cited digest.

## Network note

Serply is an authenticated API, so every call goes through `./secretcurl` with the `{SERPLY_API_KEY}` placeholder. Never rewrite the placeholder to `$SERPLY_API_KEY`, and never print the key.

## Phase 1 — Search

Stop early if the key is missing:

```bash
[ -n "${SERPLY_API_KEY:-}" ] || { echo "SERPLY_API_KEY is required for this skill"; exit 1; }
```

Build the request. `SERPLY_NEWS=1` switches to Google News results, `SERPLY_GL` sets the country (for example `us`, `de`), and `SERPLY_FRESHNESS` limits results to the past `day`, `week`, `month`, or `year`:

```bash
QUERY="${var:-current notable developments in AI and technology}"
URL="https://api.serply.io/v1/search?q=$(printf '%s' "$QUERY" | jq -sRr @uri)&num=10"

[ "${SERPLY_NEWS:-}" = "1" ] && URL="$URL&tbm=nws"
[ -n "${SERPLY_GL:-}" ] && URL="$URL&gl=$(printf '%s' "$SERPLY_GL" | jq -sRr @uri)"
case "${SERPLY_FRESHNESS:-}" in
  day)   URL="$URL&tbs=qdr:d" ;;
  week)  URL="$URL&tbs=qdr:w" ;;
  month) URL="$URL&tbs=qdr:m" ;;
  year)  URL="$URL&tbs=qdr:y" ;;
esac

HTTP=$(./secretcurl -s -o /tmp/serply-search.json -w '%{http_code}' \
  --max-time 30 "$URL" \
  -H "X-Api-Key: {SERPLY_API_KEY}" \
  -H "User-Agent: aeon")

echo "serply http=$HTTP bytes=$(wc -c </tmp/serply-search.json)"
```

On `HTTP=200`, extract the organic results. Each entry in `results[]` has `title`, `link`, `description`, and a `metadata` object with the source (`display_url`) and, when Google shows one, the age (`published_time`):

```bash
if [ "$HTTP" = "200" ] && [ -s /tmp/serply-search.json ]; then
  jq -r '.results[]? | [(.title // ""), (.link // ""), (.description // ""),
    (.metadata.display_url // ""), (.metadata.published_time // "")] | @tsv' \
    /tmp/serply-search.json > /tmp/serply-results.tsv
  RESULT_COUNT=$(wc -l < /tmp/serply-results.tsv)
  echo "results=$RESULT_COUNT"
else
  RESULT_COUNT=0
fi
```

## Phase 2 — Format

Drop entries without a title or link and collapse near-duplicates from the same domain. Keep the top 5 to 7:

```
*Serply Search — ${today}*

Query: "${var}"
Results: ${result_count}

1. **[Title](link)** (source, age)
   Snippet...

2. **[Title](link)** (source, age)
   Snippet...
```

Omit the age when `published_time` is empty. Mark the digest as news when `SERPLY_NEWS=1`.

## Phase 3 — Deliver and log

Send the digest with `./notify`.

Append to `memory/logs/${today}.md`:

```
### serply-search
- Query: "${var}"
- Mode: web | news
- Results: N found, M delivered
- Status: http=${HTTP}
```

## Error handling

Record the real reason in the log and in the notification. Do not fall back silently.

- **401**: the body is `{"detail": "..."}`; tell the operator to check `SERPLY_API_KEY` (`serply-auth-invalid`).
- **429 or 402**: out of credits or rate limited (`serply-quota`).
- **5xx or timeout**: `serply-unavailable`.
- **200 with no results**: say so and suggest a broader query (`serply-empty`).

## Environment variables

- **`SERPLY_API_KEY`** (required): API key from [serply.io](https://serply.io). New accounts get free credits; see [serply.io/docs](https://serply.io/docs).
- **`SERPLY_NEWS`** (optional): `1` for Google News results instead of web results.
- **`SERPLY_GL`** (optional): two-letter country code for localized results.
- **`SERPLY_FRESHNESS`** (optional): `day`, `week`, `month`, or `year`.

## Constraints

- Never expose the API key in logs or notifications.
- Attribute results to Serply in the digest.
- Only report links that came back in `results[]`; never invent sources.
- Schedule no more often than hourly so a run does not burn credits needlessly.
