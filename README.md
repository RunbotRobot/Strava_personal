# Strava Personal

A personal, single-page dashboard for Strava data. Built incrementally — features get added as they're needed.

## Current feature: Local Legend tracker

Shows every segment you've run in the last 90 days, alongside:

- **Mine** — how many times you've run it in that window
- **Legend** — the current local legend's effort count in that window (efforts from before the segment
  existed don't count toward either side of this comparison)
- **To Beat** — how many more runs you need to surpass them (ties don't count — you have to pass them)
- **Mi To Start** — straight-line distance from your location when the page loaded to the segment's start
- A trophy icon next to segments where you're already the legend

Click a segment name to open it on Strava in a new tab. Click any column header to sort by it. Tap the
closed-eye icon to hide a segment you don't want cluttering the list (e.g. one you're not allowed back on
for community-custody reasons) — check "Show hidden segments" to bring one back.

Activities recorded as `Velomobile` are always excluded — that type is reserved for car-recorded scouting
routes, not real efforts.

The page forces a landscape layout even if your phone's rotation lock is set to portrait (pure CSS, no
permissions needed) — turn the phone clockwise to read it right-side up.

## Setup

This is a static page — no build step, no backend of its own. It talks to Strava's API directly from your browser.

1. **Strava API app**: you can reuse the same Client ID/Secret you already created for RouteGuard at
   [strava.com/settings/api](https://www.strava.com/settings/api) — Strava's "Authorization Callback Domain"
   check only looks at the domain (e.g. `yourname.github.io`), not the path, so one app works for both.
2. **Token relay**: Strava's OAuth token endpoint can't be called directly from a browser (no CORS), so a
   tiny relay is needed to forward that one request. You can reuse the same Cloudflare Worker relay you
   already deployed for RouteGuard.
3. **Hosting**: enable GitHub Pages for this repo (Settings → Pages → deploy from `main` branch, root).
   Once it's live, open the page, paste in your Client ID / Secret / relay URL, and click **Connect to Strava**.
4. Click **Sync last 90 days**. First sync fetches details for every activity in the window (one API call
   per activity, plus one per unique segment), so it can take a bit and is deliberately paced to stay under
   Strava's rate limits. Subsequent syncs only fetch what's new.

All credentials and data are stored in your browser's `localStorage` by default — nothing leaves your device
except calls to Strava's own API and the token relay, unless you set up worker persistence (below).

**On localStorage and RouteGuard**: both apps are hosted under the same GitHub Pages account domain
(`yourname.github.io`), just at different paths (`/routeguard`, `/strava_personal`). `localStorage` is scoped
per-*origin* (scheme + host), not per-path, so the ~5–10MB quota (browser-dependent) is a single shared pool
across both apps, not a separate allowance each. This app uses an `sp_` key prefix (RouteGuard uses `rg_`) so
the two never collide directly, but RouteGuard's map cache regularly fills that shared pool and evicts its own
data to cope — see "Worker persistence" below for how this app avoids getting caught in that squeeze.

## Worker persistence (optional, recommended)

RouteGuard's tile/route cache runs into the shared localStorage quota often enough that it has its own LRU
eviction logic. That eviction only ever touches RouteGuard's own keys — it can't delete this app's data — but
once RouteGuard has filled the *entire* shared quota, this app's own `localStorage` writes can start failing
too, until RouteGuard's eviction frees some of that shared space back up. To sidestep that entirely, this app
can optionally persist its synced data server-side, on the same Cloudflare Worker used for the OAuth token
relay (`RunbotRobot/routeguard-worker`), via a `/strava-state` endpoint added there.

To turn it on:
1. In the Cloudflare dashboard, open that worker → **Settings → Variables and Secrets → Add** → name it
   `STRAVA_SYNC_SECRET`, value can be any long random string you pick. This is the only manual step —
   everything else about the endpoint is already deployed.
2. Paste that same value into this app's **Worker sync secret** field (Strava Connection panel).
3. That's it — every sync (and every hide/unhide) now also pushes the full cached state to the worker, and
   the app pulls it back from there on load, before falling back to whatever's in localStorage if the worker
   is unreachable.

Leaving the field blank keeps everything exactly as it was — local-only, no server round trip. The
`/strava-state` endpoint itself refuses all requests until `STRAVA_SYNC_SECRET` is set, so there's no
window where it's live but unprotected.

Long-term, once custom domains are in the picture, each app would get fully separate localStorage anyway —
this worker step is the practical fix for right now.

## Notes / known limitations

- Strava's public API docs don't officially document the `local_legend` field on segment details, even
  though the Strava app itself shows this data. If a segment shows "unavailable" for the legend column,
  it means that field wasn't present in the API response for that segment — open an issue (or just mention
  it) with the segment ID so the parsing can be adjusted.
- "Mi To Start" needs location permission; if you deny it or it can't get a fix, that column just shows "—".
- The forced-landscape CSS trick can only guess one physical rotation direction. If it displays upside-down
  or mirrored for you, it's a one-line sign flip to fix — just say so.
