# Spectre Nation X Scoreboard

- Page: https://htmlpreview.github.io/?https://github.com/spectre-pixel69/spectre-nation-media-kit/blob/main/scoreboard.html
- `scoreboard.html` holds no numbers. It fetches
  `https://raw.githubusercontent.com/spectre-pixel69/spectre-nation-media-kit/main/scoreboard-data.json?t=<now>`
  and re-fetches every 5 minutes. **Bots and routines only edit `scoreboard-data.json`.** Do not touch `index.html`.
- Everything derived (per-day totals, top/dead posts, by-type, best hours, replies/day, on-pace) is computed in the browser from `posts[]` and `followers`.
- Never write made-up numbers. Use `null` or leave the array empty when there's no data; the page shows an empty state.

## Schema (`schema_version: 1`)

| Key | Type | Notes |
|---|---|---|
| `last_updated` | ISO 8601 with offset | Set on every write. Shown in PT. |
| `last_updated_by` | string | e.g. `"hourly-pull"`, `"growth-bot"` |
| `account` | object | `handle`, `user_id` (1407331370825777152), `name`, `profile_url` |
| `followers.current` | int | Latest follower count |
| `followers.as_of` | ISO time | When `current` was read |
| `followers.following`, `followers.total_posts` | int | From X public_metrics |
| `followers.goal` / `goal_date` | int / `YYYY-MM-DD` | 10000 / 2026-12-31 |
| `followers.start` | `{count,date}` | Baseline for the linear on-pace line. Don't change it. |
| `followers.milestones[]` | `{count,date}` | 2500 Nov 15, 5000 Nov 29, 7500 Dec 13, 10000 Dec 31 |
| `followers.history[]` | `{date,count}` | One per PT day. Replace today's row, don't add a second one. |
| `replies_target_per_day` | int | 20 |
| `posts[]` | object | One per X post. Key is `id` (unique). Fields: `id, url, created_at` (UTC ISO), `created_pt` (`YYYY-MM-DD HH:MM`), `type` (`clip, text, photo, article, product, reply, quote, repost, live`), `text` (≤140 chars), `impressions, likes, replies, reposts, quotes, bookmarks` (int or null) |
| `bot_log[]` | object | `{time (ISO), type, text, url, post_id, status ("posted" or "failed")}` |
| `live_shows[]` | object | `{date (YYYY-MM-DD), platform, title, peak_viewers, avg_viewers, total_views, duration_min, new_followers, notes, source:"screenshot"}`. Use null for fields you don't have. |
| `not_connected[]` | `{name,status}` | Rumble, YouTube, Kick, Twitch, Product clicks = `MISSING`. Remove an entry (or set `"CONNECTED"`) when its data arrives. |
| `notes[]` | string | Shown at the bottom of the page |

## How the Growth Bot appends a post (each time a post goes out)

1. GET the file with its `sha`: `GET /repos/spectre-pixel69/spectre-nation-media-kit/contents/scoreboard-data.json`.
2. Append to `bot_log`:
   ```json
   {"time":"2026-10-25T09:00:00-07:00","type":"clip","text":"First 120 chars…","url":"https://x.com/SpectreNation1/status/<id>","post_id":"<id>","status":"posted"}
   ```
3. Also upsert into `posts[]`, matched on `id`, with `impressions` etc. set to `null` until the hourly pull fills them in.
4. Set `last_updated` and `last_updated_by:"growth-bot"`, then PUT with the `sha` (commit message `scoreboard: log post <id>`). On a 409 or sha conflict, re-GET and try again. Never rewrite the whole file from stale data.

## Hourly pull (followers and impressions)

1. `get_users_by_username(username=SpectreNation1, user.fields=public_metrics)` gives `followers.current`, `following`, `total_posts`, and `as_of=now`. Upsert today's PT row in `followers.history`.
2. `get_users_posts(id=1407331370825777152, start_time=<now-7d>, max_results=100, post.fields=created_at,public_metrics,referenced_tweets,attachments,in_reply_to_user_id,article, expansions=attachments.media_keys, media.fields=type)`. Follow `meta.next_token` until it's gone. Request only public fields: `non_public_metrics` and `organic_metrics` return authorization errors.
3. For each post, upsert into `posts[]` by `id` and overwrite the metric fields with the latest lifetime values. Keep older posts. Optionally trim to the last 30–60 days.
4. Type rules: `replied_to` ref → reply, `retweeted` → repost, `article` → article, product keywords → product, `quoted` → quote, video/gif media → clip, photo → photo, otherwise text.
5. Set `last_updated` and `last_updated_by:"hourly-pull"`, then commit to `main`. The page picks up the change within about 5 minutes (raw.githubusercontent CDN can add up to about 5 minutes).

## Live show stats (manual)
From a screenshot, append one `live_shows[]` row with only the numbers the screenshot shows. Use null for the rest.
