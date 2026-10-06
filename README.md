# Track and Thrive Replay Automation

Auto-posts each Tuesday's "Track & Thrive with Coach Emily" replay link to a
Squarespace page (suggested: bianutrition.com/tt-replays). Sibling of
`../ph-replays` (Power Hour) and `../elevation-replays`.

## How it works

1. `replays.json` is the source of truth (seeded with the first call, 2026-10-06).
2. A GitHub Action runs Tuesdays at 18:00, 18:30, 19:00, and 20:00 UTC (covers 2 PM ET
   year-round regardless of daylight saving, plus built-in retries). The call itself
   runs Tuesdays 12:00-1:00 PM ET, so the first slot lands an hour after it ends.
3. The script POSTs to the Grain v2 API for a recording from the **most recent Tuesday
   (ET)** whose title contains both "Track" and "Thrive" (the real title is
   "Track & Thrive with Coach Emily"), pulls its `recording_url` share link, and
   appends `{date, url}` to the JSON. "Most recent Tuesday" rather than "today" because
   GitHub fires cron on low-traffic repos hours late.
4. The workflow commits the change; the Squarespace code block fetches the raw JSON
   and renders the list. Page updates within ~5 minutes of the commit (GitHub raw CDN cache).

The script is idempotent: if that Tuesday's entry already exists, later runs exit quietly.
Extra cron slots are free retries in case Grain hasn't finished processing by 2 PM.

## Stream Deck button (manual run)

`workflow_dispatch` runs (the Stream Deck button in `../streamdeck-actions`, or
"Run workflow" in the Actions tab) set `MANUAL=true`, which changes two things:

- It looks for the newest call in the **last 6 days** instead of the most recent
  Tuesday, so a press any time from Tuesday afternoon through the weekend works.
- "Nothing found" is a **failure** (exit 1, Slack `:hourglass:` note) instead of a
  quiet no-op. That's deliberate: the `check-manual-run` job makes the scheduled
  slots stand down when a *successful* manual run happened in the last 12 hours,
  so a press that came before Grain finished processing must not count.

Two layers stop a double run after a button press:

1. `check-manual-run` skips every scheduled slot for 12h after a successful press.
2. Even past 12h, a scheduled run bails with "Already posted" when that Tuesday's
   date is already in `replays.json`.

Every run ends with a `RESULT: …` line.

## Setup (one time)

Repo lives at **https://github.com/MattBia/track-thrive-replays** (public, so the
Squarespace fetch works). Three steps remain, two of them needing secrets only Matt has:

1. Repo → Settings → Secrets and variables → Actions → New repository secret:
   - `GRAIN_API_TOKEN_V2` — the same value as in `power-hour-replays`. **Required.**
   - `SLACK_WEBHOOK_URL` — same value as in `power-hour-replays`. Optional.
2. Create the Squarespace page (suggested slug `/tt-replays`), add a Code Block, and
   paste the whole of `squarespace-embed.html`. If a different slug is used, update
   `PAGE_URL` in `scripts/update_replays.py` so Slack notes point at the right page.
3. Test: Actions tab → "Track and Thrive Replay Updater" → Run workflow. Expected
   today: "already on the page and Grain has nothing newer".

## Notes / failure modes

- If the recording title doesn't contain both "Track" and "Thrive", nothing posts and
  (if Slack is configured) you get pinged after the final 4 PM attempt. Manual
  fallback: add a line to `replays.json` and commit.
- If the recording isn't shared publicly in Grain, the script fails with a clear
  message rather than posting a dead link.
- The JSON contains only share links that are already public on the website, so
  nothing sensitive is exposed by the public repo.
