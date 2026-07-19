# Strava Personal

A personal, single-page dashboard for Strava data. Built incrementally — features get added as they're needed.

## Current feature: Local Legend tracker

Shows every segment you've run in the last 90 days, alongside:

- **My Efforts** — how many times you've run it in that window
- **Local Legend** — who currently holds it, and their effort count
- **Runs to Beat** — how many more runs you need to surpass them (ties don't count — you have to pass them)
- A trophy icon next to segments where you're already the legend

Click a segment name to open it on Strava in a new tab. Click any column header to sort by it.

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

All credentials and data are stored only in your browser's `localStorage` — nothing leaves your device except
calls to Strava's own API and the token relay.

## Notes / known limitations

- Strava's public API docs don't officially document the `local_legend` field on segment details, even
  though the Strava app itself shows this data. If a segment shows "unavailable" for the legend column,
  it means that field wasn't present in the API response for that segment — open an issue (or just mention
  it) with the segment ID so the parsing can be adjusted.
- The "run" filter defaults to activity types containing "Run" (covers `Run` and `TrailRun`). Checkboxes
  let you include other activity types the moment you need segment tracking for rides, walks, etc.
