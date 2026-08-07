# Orange Robot Store

Purchase flow for The Orange Robot (buy.html, admin-orders.html, admin-settings.html, order-status.html), served by GitHub Pages from this repo (`WebMaven50/OrangeRobotStore`). Backed by the `orange-robot-store` Cloudflare Worker.

## Multi-machine workflow — read this first

This repo (and its Worker) gets worked on from more than one machine (at least a Mac mini and a MacBook Pro), sometimes in parallel Claude Code sessions that don't know about each other. There is no other coordination mechanism, so:

- **Before starting any work on the HTML pages**, run `git fetch origin && git log HEAD..origin/main --oneline`. If that shows commits, `git pull` and actually read the diffs before assuming you know the current state of any file.
- **Before touching the Worker (`orange-robot-store`), always pull the live *deployed* source fresh** — see below. Do not trust a locally cached copy from earlier in the same conversation; the other machine may have deployed since then, and the Worker has no git history of its own to check against.
- **After finishing a change and getting confirmation to ship it**, commit + push the HTML changes and `wrangler deploy` the Worker right away — don't leave work uncommitted/undeployed across a session boundary.
- If you find unexpected commits or Worker behavior you have no memory of producing, that's the other machine, not a bug — read everything before touching anything nearby.

## Pulling the live Worker source (it has no git history)

The Worker was never scaffolded with `wrangler init` in a tracked project, so the deployed script is the only source of truth. Pull it directly from the Cloudflare API before editing:

```bash
TOKEN=$(node -e "
const fs=require('fs');
const toml=fs.readFileSync('/Users/victorgelfo/Library/Preferences/.wrangler/config/default.toml','utf8');
const m=toml.match(/oauth_token\s*=\s*\"([^\"]+)\"/);
console.log(m ? m[1] : '');
")
ACCOUNT_ID="cb4c4cc98d136e028b55d93b37356c2f"
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/scripts/orange-robot-store" \
  -o /tmp/worker-raw.bin
# response is multipart/form-data; extract the "index.js" part before editing
```

Then edit, `node --check` for syntax, test the changed logic standalone against extracted code blocks, `wrangler deploy --dry-run` to confirm bindings, get confirmation, `wrangler deploy`, then verify live with curl.

`wrangler.jsonc` for this Worker (bindings, KV namespace, R2 bucket, cron) isn't committed anywhere — reconstruct it from the account settings (`.../workers/scripts/orange-robot-store/settings`) and schedules (`.../schedules`) endpoints if starting a fresh scratch directory.

## Current payment/delivery model

Digital delivery is **gated, not honor-system**: a transaction ID is required at checkout, and `/download/epub` only releases the file once `order.status === "paid"` or `order.mempoolSeen` (the transaction is visible on the network with the right amount, even before confirmation) — checked server-side, not just hidden client-side. Both checks reuse the same anti-replay protection (a txid can't be claimed by more than one order). Physical orders still require full on-chain confirmation (`"paid"`) before fulfillment.

## Infrastructure

- **R2 bucket** `orange-robot-files` — stores `the-orange-robot.epub`, served via the Worker (not a static file in this repo anymore — don't re-add it). Swappable from `admin-settings.html` without a git push.
- **KV namespace** `ORDERS` (binding `ORDERS`) — orders, settings, download counters/log, rate-limit state, admin lockout state.
- **Secrets** (write-only, never viewable again once set): `ADMIN_PASSWORD`, `RESEND_API_KEY`. **Plain vars**: `NOTIFY_EMAIL` (visible to anyone with Cloudflare dashboard access).
- CORS on the Worker is locked to `https://webmaven50.github.io` — the only legitimate frontend origin. Local testing from `file://` or `localhost` will show fetches failing; that's expected, not a bug.
- Every HTML page carries a `<meta http-equiv="Content-Security-Policy">` tag (GitHub Pages can't send real HTTP headers). A new external host being fetched/loaded needs adding to that page's CSP `connect-src`/`script-src`/etc. or the browser will silently block it.
