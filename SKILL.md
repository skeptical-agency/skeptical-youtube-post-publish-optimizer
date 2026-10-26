---
name: skeptical-youtube-post-publish-optimizer
description: "Automate post-publish YouTube metadata with a nightly pass: find just-published videos, push tags, category, recording date and API-only defaults, verify every write by read-back, and run alert-only QA gates and a packaging-change protocol. Use when a scheduler or repurposing tool cannot set tags, category or recording date, or before you start swapping titles and thumbnails."
when-to-use: "You publish shorts or videos through a scheduler or repurposing tool (OpusClip, Repurpose.io, etc.) that cannot set tags, category, recording date or API-only defaults; you want an agent or cron job that keeps metadata fresh, verifies every write and only alerts on audio problems, retention drops or packaging failures; or you are about to start swapping titles or thumbnails on underperforming videos."
---

# YouTube post-publish optimizer

## What this does (plain English)

When you publish YouTube videos through a scheduling or repurposing tool, some settings can only be set through the YouTube Data API after the video is live: tags, category, recording date and a few standing defaults. This skill teaches your AI assistant to do that work for you, every night, as one short pass: find videos that were just published, build the tags (from keyword research, with a fallback if the research tool is out of credits), push the metadata with one update per video, read it back to confirm it really landed, and send you a short plain-text report. You do not need to have used the YouTube Data API before. The skill spells out the error handling (what to retry and what never to retry), the rules that keep one channel's login away from another, and a cost trick that replaces constant polling with one gated daily run.

It also contains the discipline rules that stop panic changes. The automation owns tags, category, recording date and standing defaults. It never owns titles or descriptions, and it never swaps a title or thumbnail by itself: it can alert and propose, and a human decides. There is no approval step for the metadata it owns, but there is always a human decision for packaging. See the README for that distinction.

It is a workflow skill, so it works the same on macOS, Linux and Windows. What you need: Python 3 and a YouTube Data API OAuth client (created in a Google Cloud project with the YouTube Data API v3 enabled). The spec does not name an install command or a library, so none is given here.

## Responsible use

The automation must never auto-delete, auto-privatize or auto-swap packaging (titles or thumbnails) on a video. The QA gates (audio verification, day 7 retention and CTR review) only alert; a human decides pulls and swaps, because false positives happen. Only manage channels you own or are authorized to manage. Keep each channel's OAuth tokens separate and private. You are responsible for following the YouTube API Terms of Service and for your API quota use. Respect each channel's banned-topics list. OpusClip, Repurpose.io and VidIQ are mentioned only as examples of tools; their trademarks belong to their owners, and this skill is not affiliated with or endorsed by them.

## Install what you need

Nothing to install for the workflow itself. It works on macOS, Linux and Windows. You need:

- Python 3 (from python.org or your system package manager)
- A Google Cloud project with the YouTube Data API v3 enabled
- A YouTube Data API OAuth client for the channel you own

## Worked example

ILLUSTRATIVE ONLY. This is one nightly pass, built from the pipeline steps and values below. All IDs are placeholders (`<VIDEO_ID>`, `<topic phrase>`, `<publish time>`), and no real channel, metric or result is shown.

1. **FIND.** Query the YouTube Data API (`search.list` with `forMine=true` and `type=video`, or `playlistItems.list` on your uploads playlist then `videos.list` by id, published within 48-72h) and diff against the state file map of already-optimized video IDs. If nothing is new, exit silently with no state write. Here, one new video is found: `<VIDEO_ID>`.
2. **AUDIO VERIFY** (optional, alert-only). Compare the published audio with your source render. If it fails, make that the first line of the report, still apply metadata, flag the video once in state, and never delete or privatize it.
3. **RETENTION and CTR REVIEW.** This runs at day 7, so a brand-new video is not reviewed yet. Nothing to do for `<VIDEO_ID>` on this pass.
4. **BUILD.** Run keyword research on `<topic phrase>`. Check your credit balance first. If the research API is dry, fall back to title-derived tags. Keep at most 15 tags, each <=30 chars, filtered against the banned-topic list. Pick the category for the content domain. Add the standing defaults: recordingDate = `<publish time>` (ISO 8601 UTC), license = youtube, madeForKids = false (sent as `status.selfDeclaredMadeForKids`), embeddable = true. Do not send location.
5. **PUSH and VERIFY.** First read the current video with `videos.list`, so you can send back everything you are not changing (the `videos.update` docs say unspecified existing values are deleted). Make one `videos.update` call for `<VIDEO_ID>`, listing the parts you are writing, with `snippet.title` and `snippet.categoryId` included whenever you update the snippet. Read it back and confirm the tags and category landed. If the read-back looks stale, re-get after ~10 seconds instead of writing again. On HTTP 412 preconditionFailed, retry with 8/16/24s backoff. On 403 quotaExceeded, do not retry: park the video in `quota_pending` and report once. Mark the video optimized in state only after the verified write.
6. **SPONSOR blocks.** None pending, so nothing to apply.
7. **REPORT.** Short plain text, for example:

```
Optimized: <VIDEO_ID> (tags, category, recordingDate, license, madeForKids, embeddable). Read-back verified.
Alerts: none.
Failures: none.
```

The video is now marked optimized in state, and it is never touched again. Its title and thumbnail are left alone: no packaging change is proposed before 72 hours.

**API notes (added by Skeptical, verified against YouTube Data API docs on 2026-10-26):** `forMine` is a `search.list` filter (with `type=video`), not a `videos.list` filter, and `videos.update` removes existing values you leave out of the parts you list, so always read first and send back what you are not changing. The full notes, with field locations, quota costs and doc links, are at the end of this file.

---

The full production recipe follows.


## When to Use

- You publish shorts or videos through a scheduler/repurposing tool (OpusClip, Repurpose.io, etc.) that cannot set tags, category, recording date, or API-only defaults, and you want those applied automatically after publish
- You want an agent or cron job that keeps metadata fresh, verifies every write, and alerts (never auto-acts) on audio problems, retention drops, or packaging failures
- You are about to start swapping titles/thumbnails on underperforming videos and need the discipline rules that prevent panic changes

Core doctrine: the optimizer OWNS tags, category, recording date, and standing defaults. It NEVER owns titles or descriptions (those are written at schedule time), and it NEVER auto-swaps packaging. Verification by read-back after every write. Alert-only QA gates; a human decides pulls and swaps.

## Pipeline (one pass per run)

1. FIND newly published videos. Query your publishing tool's scheduled-posts API or the YouTube Data API (`search.list` with `forMine=true` and `type=video`, or `playlistItems.list` on your uploads playlist then `videos.list` by id, published within 48-72h) **[Skeptical correction: the original text here read "videos.list, forMine". See the API notes at the end of this file.]**. Diff against a state file map of already-optimized video IDs. Nothing new = exit silently, no state write.
2. AUDIO VERIFY (alert-only, optional but recommended for TTS/AI-voiced content). Compare the PUBLISHED YouTube audio against your source render: speaker-embedding similarity vs a calibrated voice bank, dropout scan, platform-vs-source diff. Platform ingest can mangle audio that the pre-publish gate passed. On FAIL: alert as the first line of the report, still apply metadata, flag the video once in state. Never auto-delete or auto-privatize; false positives happen and a human must listen.
3. RETENTION & CTR REVIEW at day 7 (alert-only). Thresholds that work: average percentage viewed < 55% on 45-55s shorts = HOOK/PACING flag (healthy is 60-75%; the fix belongs to the NEXT video: rehook at 0:20-0:30, tighten the first 30s). CTR < 0.6x the window mean with >2,000 impressions = PACKAGING flag. Review each video once; track reviewed IDs in state.
4. BUILD the metadata set. Tags: run keyword research on the video's topic phrase (VidIQ keyword_research or equivalent); check your credit balance first and fall back to title-derived tags when the research API is dry. Max 15 tags, each <=30 chars, filtered against your channel's banned-topic list. Category: pick per content domain and default it. Standing defaults on EVERY video: recordingDate = publish time (ISO 8601 UTC), license = youtube, madeForKids = false, embeddable = true. Do NOT send location (silently discarded by the API).
5. PUSH + VERIFY. One `videos.update` call per video. Read back after writing and confirm tags/category actually landed; a stale read-back means re-get after ~10 seconds, not a second write. Error handling: HTTP 412 preconditionFailed is transient, retry with 8/16/24s backoff; 403 quotaExceeded must NOT be retried, park the video in a `quota_pending` state list and report once. Mark the video optimized in state ONLY after a verified write.
6. SPONSOR/DESCRIPTION BLOCKS (optional). Keep `pending_sponsor_*.json` files with a title-match regex + publish window; on match, prepend the block to the existing description (preserve everything else exactly), post the companion comment (the API cannot pin comments; tell the human to pin in Studio), verify both landed, then rename the file to `done_*` so it can never apply twice.
7. REPORT. Short plain text: what was optimized, every alert, every failure. Honest reporting beats pretty reporting.

## Keyword-research tandem (pre vs post publish)

- PRE-publish: keyword research and title scoring belong to the BUILD step (topic validation, title generation, scored options). Titles reaching the optimizer are already optimized; the optimizer never changes them.
- POST-publish: the optimizer consumes keyword research for TAGS only, with a balance gate and a title-word fallback so the pipeline never stalls on a dead API.
- Post-publish title/thumbnail decisions follow the packaging protocol below, NOT keyword scores.

## Packaging change protocol

- NEVER propose title/thumbnail changes on a video younger than 72 hours. The first algorithm test cycle takes ~3 days (evergreen up to a month); early CTR is noise, not signal.
- Time-sensitive content after 72h: judge in 24-hour blocks. Impressions stalled AND CTR below 0.6x channel average = propose a thumbnail swap in the next block, title the block after. One week maximum, then accept the result and move on.
- After stabilization, CTR is not an action metric; long-tail videos routinely go viral at 3% CTR. The only actionable CTR for an established channel is first-24h vs its own average, answering "did the packaging fail the core fans?" If packaging was strong and views are weak, it is a CONCEPT problem and repackaging will not save it; say so in the report.
- Shorts thumbnail swap = re-grab a better hook frame from the video itself (`thumbnails.set`). Keep long-form thumbnails on your real design pipeline.
- The automation NEVER auto-swaps packaging. Propose, human approves.

## Hard rules

- NO BROWSER, EVER: terminal scripts and direct HTTP/API clients only. Anything that seems to need a browser (cloud console, OAuth consent screens) gets reported, not clicked.
- CHANNEL ISOLATION for multi-channel setups: each channel gets its own OAuth token home, and every API call for that channel is prefixed with the token-home override. Never let a job authenticated as channel A touch channel B; verify the video belongs to the expected channel before writing.
- CONTENT FILTERS: tags and metadata must respect the channel's banned-topics list. When in doubt, exclude.
- Preserve existing title/description outside the sponsor-block flow. One update call per video. Never re-touch a video already marked optimized in state.
- OAuth dead (refresh failed / not authenticated): first report line is a PLATFORM ALERT with the exact re-auth command for the human, then stop. Do not silently skip.

## Cost discipline

A 15-minute polling schedule for this workflow once consumed ~37% of an agent's total token spend. Fix: run once daily and add a cheap monitor script (a few API calls, no LLM) that gates waking the full agent only when new video IDs exist. Never solve latency by raising poll frequency; tighten the monitor gate instead.

## Extending to a new channel

1. New state directory with its own `state.json` (optimized-IDs map + topics log).
2. Separate OAuth token home; never share tokens between channels.
3. Clone an existing job prompt, swap channel ID, category default, and tag seed domains.
4. Pin the agent model explicitly so a model swap cannot silently change behavior.
5. First run: verify the read-back manually before trusting the state marks.


## API notes (added by Skeptical, verified against YouTube Data API docs on 2026-10-26)

These notes are not part of the original spec. They were checked against the official YouTube Data API reference pages listed here on 2026-10-26. Where the docs do not confirm something, it is left out. The spec rules above are unchanged, except for the one marked correction in Pipeline step 1.

### Finding new videos (step 1)

- `forMine` is a `search.list` filter, not a `videos.list` filter. The documented `videos.list` filters are `chart`, `id` and `myRating`. With `forMine=true`, `type` must also be `video`, and these cannot be set in the same request: `videoDefinition`, `videoDimension`, `videoDuration`, `videoEmbeddable`, `videoLicense`, `videoPaidProductPlacement`, `videoSyndicated`, `videoType`. The request must be authorized.
- The `search.list` docs say that for `order=date`, results rely on the search index, so you may see indexing delays or incomplete result sets, and that to reliably retrieve a channel's most recent uploads you should use `playlistItems.list` on the channel's uploads playlist instead.
- Uploads-playlist route: `channels.list` with `mine=true` and `part=contentDetails` returns `contentDetails.relatedPlaylists.uploads`. Pass that as `playlistId` to `playlistItems.list` (the docs' own sample reads each video's ID from `snippet.resourceId.videoId`). Then call `videos.list` with `id` for the details you need.
- Quota costs as listed in the docs: `channels.list` 1 unit, `playlistItems.list` 1 unit, `videos.list` 1 unit, `videos.update` 50 units, `thumbnails.set` 50 units. `search.list` has its own bucket: a default of 100 calls per day, and each call costs 1 in that bucket. Other endpoints share a default of 10,000 units per day, daily quota resets at midnight Pacific Time, and every request, including an invalid one, costs at least 1. Each extra page of results is another call.

### Writing metadata (steps 4 and 5)

- `part` on `videos.update` lists the parts you are writing. The docs say the method overrides the existing values of all mutable properties in any part you list. If you list a part and leave a property out, its existing value is removed (the docs' own example is `status.privacyStatus`, which reverts to the default). So read the current video first with `videos.list`, change only what you mean to change, and send the rest back unchanged (read-modify-write).
- If you update the `snippet` part, `snippet.title` and `snippet.categoryId` are required, along with the video `id`. To keep the title and description as the spec requires, send the existing `snippet.title` and `snippet.description` back, and any other snippet values you want to keep, such as existing `snippet.tags[]` and `snippet.defaultLanguage`. The docs say a property that has a value but is not specified in the update is deleted. The exceptions are `status.selfDeclaredMadeForKids` and `status.containsSyntheticMedia`, which only change when explicitly set.
- Field locations: tags are `snippet.tags[]`; category is `snippet.categoryId`; the recording date is `recordingDetails.recordingDate` (ISO 8601, `YYYY-MM-DDThh:mm:ss.sssZ`); the license is `status.license` (`creativeCommon` or `youtube`); embeddable is `status.embeddable`.
- "madeForKids": `status.madeForKids` reports the video's current made-for-kids status. The property you set in `videos.update` is `status.selfDeclaredMadeForKids`.
- The docs also limit `snippet.tags[]` to 500 characters in total, counting the commas between tags, and counting quotation marks around any tag that contains a space. That is in addition to the spec's own limits (max 15 tags, each <=30 chars).
- Location: the docs mark `recordingDetails.location` and its latitude, longitude and altitude as deprecated, and the list of properties you can set in `videos.update` does not include it. The docs do not say it is silently discarded. The spec's rule "Do NOT send location" is consistent with the deprecation.

### Sources

- https://developers.google.com/youtube/v3/docs/search/list
- https://developers.google.com/youtube/v3/docs/channels/list
- https://developers.google.com/youtube/v3/docs/playlistItems/list
- https://developers.google.com/youtube/v3/docs/videos/list
- https://developers.google.com/youtube/v3/docs/videos/update
- https://developers.google.com/youtube/v3/docs/videos (resource representation)
- https://developers.google.com/youtube/v3/determine_quota_cost
