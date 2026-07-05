# idea-scout

Daily Claude routine that mines live demand signals and produces 0-6 evidence-backed build ideas per day (3-4 week solo builds, agent-heavy, integration-biased). Sunday run is a weekly review that surfaces the top 3 ideas for human validation.

## Files
- `ROUTINE-PROMPT.md`: the prompt to paste into the routine (fill the {{DELIVERY}} placeholder first)
- `SOURCES.md`: verified endpoint runbook the routine follows (all endpoints live-tested 2026-07-05)
- `DOMAINS.md`: the 12-domain superset with query seeds (edit to taste)
- `LEDGER.json`: cross-run memory (clusters, rotation, dedupe); the routine maintains it on branch `claude/ledger`

## Setup checklist
1. Create a private GitHub repo `idea-scout` with these files on `main` (remote via SSH alias, e.g. `git@github-vicky81125:...`).
2. On claude.ai > Code > environments: create/edit the environment for this repo:
   - Network access: **Full** (or Custom allowlist: hn.algolia.com, api.github.com, trends.google.com, api.stackexchange.com, freelancer.com, itunes.apple.com, api.npmjs.org, pypistats.org, r.jina.ai, api.exa.ai, reddit.com, oauth.reddit.com)
   - Env vars: `EXA_API_KEY` (required), `JINA_API_KEY` (free key, recommended), optional `REDDIT_CLIENT_ID`/`REDDIT_CLIENT_SECRET` (free script app from reddit.com/prefs/apps), optional `GITHUB_TOKEN`
   - Note: env vars are visible to anyone who can edit the environment (no secrets store yet)
3. Create the routine: repo = this repo (vicky81125/ideas), schedule = daily 01:00 UTC (6:30 AM IST), cron `0 1 * * *`, prompt = ROUTINE-PROMPT.md contents, connectors = keep Slack (delivery channel #ideas-and-research), remove the rest.
4. First run: trigger once manually, read the transcript end to end, tune DOMAINS.md seeds and the kill threshold if needed.

## Operating notes
- Runs bill against the Claude subscription (no separate compute charge); routines have a per-account daily run cap; min interval 1 hour.
- A green run status only means no infrastructure error; read the report, not the status.
- No auto-retry on failure; a failed day is just skipped.
- Expected cost outside the subscription: a few cents/day of Exa (~15 searches/run at $7 per 1k). Everything else is free-tier.
- Review cadence: run it for 4-5 weeks; the Sunday weekly review is the artifact that matters. The routine ranks and evidences; the human validates (landing page, DMs into source threads, concierge MVP).
