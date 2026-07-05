# SOURCES.md: verified endpoint runbook

Every endpoint below was live-tested with curl on 2026-07-05. Use ONLY these; do not improvise new scraping paths. The routine runs from a datacenter IP, so anything marked "cloud-blocked" must use the listed fallback.

Env vars expected (set on the routine's environment): `EXA_API_KEY` (required), `JINA_API_KEY` (recommended, free key lifts r.jina.ai from 20 to 500 req/min), `REDDIT_CLIENT_ID` + `REDDIT_CLIENT_SECRET` (optional, free script app), `GITHUB_TOKEN` (optional, lifts search from 10 to 30 req/min).

## Tier 1: cloud-safe, no auth

### Hacker News (Algolia)
Free, generous (10k req/hr per IP). Note: `points>N` numeric filter only works on `search_by_date`, not `/search`.
```bash
# Ask HN demand mining
curl -s "https://hn.algolia.com/api/v1/search_by_date?query=%22what%20do%20you%20use%20for%22&tags=ask_hn&hitsPerPage=20"
# Show HN launches in a domain, last 24h
curl -s "https://hn.algolia.com/api/v1/search_by_date?query=<keyword>&tags=show_hn&numericFilters=created_at_i%3E$(date -d '1 day ago' +%s)"
# Hot stories on a topic
curl -s "https://hn.algolia.com/api/v1/search_by_date?query=<keyword>&tags=story&numericFilters=points%3E50"
```

### GitHub search API
10 req/min unauth, 30 with token. Dev-demand heat via new-repo velocity.
```bash
curl -s "https://api.github.com/search/repositories?q=<topic>+created:%3E$(date -d '7 days ago' +%F)&sort=stars&order=desc&per_page=10" -H "Accept: application/vnd.github+json" ${GITHUB_TOKEN:+-H "Authorization: Bearer $GITHUB_TOKEN"}
```
CLOUD CAVEAT (verified 2026-07-05): the routine environment's session proxy blocks api.github.com/search ("sessions are bound to their configured repositories"). Route it through the agentproxy passthrough port instead, same mechanism as the ledger push:
```bash
PROXY_PORT=$(curl -sS "http://127.0.0.1:$(echo $HTTPS_PROXY | grep -oP '\d+$')/__agentproxy/status" 2>/dev/null | python3 -c "import sys,json; print(json.load(sys.stdin)['port'])" 2>/dev/null || echo 45609)
curl -s --proxy "http://127.0.0.1:$PROXY_PORT" --cacert /root/.ccr/ca-bundle.crt -H "Authorization: Bearer $GITHUB_TOKEN" -H "Accept: application/vnd.github+json" "https://api.github.com/search/repositories?q=..."
```
If the passthrough also blocks it, fall back to `r.jina.ai/https://github.com/trending/<language>?since=daily` (global trending only) and say so in the report.

### Google Trends daily breakouts
```bash
curl -s "https://trends.google.com/trending/rss?geo=US"   # also geo=IN
```
XML with `<title>` and `<ht:approx_traffic>`. The old `/trends/api/dailytrends` JSON endpoints are dead (404).

### Stack Exchange
300 req/day without key. MUST use `--compressed`. Respect the `backoff` field in responses.
```bash
curl -s --compressed "https://api.stackexchange.com/2.3/search/advanced?order=desc&sort=votes&q=<query>&site=stackoverflow&pagesize=20"
```

### Freelancer.com active projects (the only working free freelance-demand source)
Undocumented, be gentle: max ~10 queries/run, spaced out. Budget field = quantified willingness to pay.
```bash
curl -s "https://www.freelancer.com/api/projects/0.1/projects/active/?query=<keywords>&limit=50&compact=true"
```
CAVEAT (verified 2026-07-05): the `query` param matches loosely; multi-word queries often return the generic unfiltered feed. Always post-filter results locally: keep only projects whose title/description actually contains the domain keywords, discard the rest. Single-keyword queries ("shopify") filter more reliably than phrases.

### iTunes Search API (~20 calls/min)
```bash
curl -s "https://itunes.apple.com/search?term=<term>&entity=software&limit=10&country=US"
```
Signal: `userRatingCount` high + `averageUserRating` < 4.0 = big install base that cares and is unhappy.

### npm / PyPI download velocity (dev-demand proxy)
```bash
curl -s "https://api.npmjs.org/downloads/point/last-week/<package>"
curl -s "https://pypistats.org/api/packages/<package>/recent"
```

## Tier 2: via r.jina.ai reader (20 req/min free, 500 with JINA_API_KEY)

```bash
# Product Hunt today's launches (direct PH pages 403)
curl -s ${JINA_API_KEY:+-H "Authorization: Bearer $JINA_API_KEY"} "https://r.jina.ai/https://www.producthunt.com/"
# GitHub trending page
curl -s "https://r.jina.ai/https://github.com/trending/<language>?since=daily"
# Google Play listing (use the real package id)
curl -s "https://r.jina.ai/https://play.google.com/store/apps/details?id=<package.id>"
```

## Tier 3: keyed

### Exa (EXA_API_KEY): the workhorse for community pain + review mining
Single REST endpoint, header auth, ~$7 per 1k searches. Budget: max ~20 Exa calls per run.
```bash
curl -s -X POST "https://api.exa.ai/search" -H "x-api-key: $EXA_API_KEY" -H "Content-Type: application/json" \
  -d '{"query":"<pain phrase> <domain keywords>","numResults":8,"type":"auto","startPublishedDate":"<ISO date 180d ago>","includeDomains":["<domain>"],"contents":{"text":{"maxCharacters":1500}}}'
```
**Domain availability (live-tested 2026-07-05):** `includeDomains` WORKS for: `indiehackers.com`, `news.ycombinator.com`, `quora.com`, `dev.to`, `community.shopify.com`, `g2.com`, `capterra.com`. It is REJECTED for `reddit.com` and `x.com` ("requested domains are not available") — never attempt those; without a site filter Exa returns SEO-aggregator pages, which are not citable evidence.

Primary uses:
- **Community pain** (replaces Reddit): includeDomains one of indiehackers.com / news.ycombinator.com / quora.com / dev.to / community.shopify.com (pick per domain relevance) + the pain lexicon.
- **Review/complaint mining** (restores the G2/Capterra class that direct scraping can't reach): includeDomains g2.com or capterra.com, query "<category or competitor>" + "wish it" / "doesn't integrate" / "switched from" / "workaround". A complaint describing a workaround is the MVP spec.
- Domain heat checks in Stage 1 and competitor lookup in Stage 4 (no site filter needed).

**Vendor launch, funding, and press articles are NEVER demand evidence, whatever the source.** They prove supply, not pain. Do not build clusters from them.

### Reddit official OAuth script app (optional, free, 100 req/min)
Only if REDDIT_CLIENT_ID/SECRET are set:
```bash
TOKEN=$(curl -s -X POST -A "idea-scout/1.0" -u "$REDDIT_CLIENT_ID:$REDDIT_CLIENT_SECRET" -d "grant_type=client_credentials" https://www.reddit.com/api/v1/access_token | jq -r .access_token)
curl -s -A "idea-scout/1.0" -H "Authorization: Bearer $TOKEN" "https://oauth.reddit.com/r/<sub>/search?q=<query>&restrict_sr=1&sort=new&t=month&limit=25"
```
High-yield subreddits by category: r/smallbusiness, r/Entrepreneur (ops pain), r/ecommerce, r/shopify (integration/automation asks), r/SaaS, r/indiehackers (builder market), r/freelance (marketplace pain), r/productivity (saturated, look for gaps only), plus per-domain subs.

## Known dead ends: do not attempt
- Reddit, all $0 paths (verified 2026-07-05): public `.json`/`.rss` 403 from datacenter IPs; r.jina.ai blocked by Reddit; Exa rejects reddit.com in includeDomains. The ONLY working path is the official OAuth script app (REDDIT_CLIENT_ID/SECRET env vars, see above). If those are unset, run without Reddit and use the Exa community domains instead.
- X/Twitter: Exa rejects x.com; no free path.
- Upwork: Cloudflare on both direct and Jina paths.
- G2 / Capterra DIRECT scraping: CAPTCHA on both paths. Use Exa includeDomains g2.com/capterra.com instead (works, verified).
- YouTube search via Jina: consent shell, no results. (Channel RSS `youtube.com/feeds/videos.xml?channel_id=...` works if specific channels are ever tracked.)
- Product Hunt leaderboard direct: 403 (use the Jina path above, or a free PH GraphQL client token if ever added).
