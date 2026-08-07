# Orange Robot Store (legacy — this repo is redirects only)

The real store now lives at `theorangerobot.com/store/` in the `WebMaven50/Orange` repo (main site repo), including the `orange-robot-store` Worker's development notes and data model. **Read that repo's `CLAUDE.md` instead of this one for anything about building the store.**

This repo now only holds lightweight redirect stub pages (`buy.html`, `admin-orders.html`, `admin-settings.html`, `admin-downloads.html`, `order-status.html`, `redeem.html`) that forward `webmaven50.github.io/OrangeRobotStore/*` traffic to the new `theorangerobot.com/store/*` URLs, preserving the query string (needed for `order-status.html?order=...` links already sent in real customer emails).

**Do not delete this repo or these stub pages** — old bookmarks and past confirmation emails point here. **Do not add new functionality here** — build it in `WebMaven50/Orange`'s `/store/` folder instead.

Each stub is a static file: a `<meta http-equiv="refresh" content="1; url=...">` fallback plus a JS `window.location.replace()` that appends `window.location.search` to the target URL. The 1-second delay (not 0) is intentional — it gives the JS redirect (which preserves the query string) time to fire before the meta-refresh (which doesn't) would otherwise win the race.
