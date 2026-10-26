# skeptical-youtube-post-publish-optimizer (Grok skill)

A nightly workflow that applies tags, category, recording date and standing defaults to just-published YouTube videos, verifies every write by read-back, runs alert-only QA gates, and enforces a 72 hour packaging-change protocol.

## Install

This repo is the skill folder: `SKILL.md` sits at the top level. Clone it into a skills directory.

User scope (all your projects), macOS or Linux:

    mkdir -p ~/.grok/skills
    git clone https://github.com/skeptical-agency/skeptical-youtube-post-publish-optimizer.git ~/.grok/skills/skeptical-youtube-post-publish-optimizer

User scope, Windows PowerShell (assumes Grok reads skills from `$HOME\.grok\skills`):

    New-Item -ItemType Directory -Force "$HOME\.grok\skills"
    git clone https://github.com/skeptical-agency/skeptical-youtube-post-publish-optimizer.git "$HOME\.grok\skills\skeptical-youtube-post-publish-optimizer"

Project scope (shared via git):

    mkdir -p .grok/skills
    git clone https://github.com/skeptical-agency/skeptical-youtube-post-publish-optimizer.git .grok/skills/skeptical-youtube-post-publish-optimizer

No git? Download the repo as a zip from GitHub and unzip it into a folder named `skeptical-youtube-post-publish-optimizer` inside the skills directory.

More skills: [skeptical-agency/grok-skills](https://github.com/skeptical-agency/grok-skills).

## Requirements

- Python 3
- A Google Cloud project with the YouTube Data API v3 enabled, and a YouTube Data API OAuth client for a channel you own
- Works on macOS, Linux and Windows. The spec names no install command or library, so none is given.

## Use

Ask Grok things like:

> run tonight's post-publish pass for my channel and report what was optimized
> set up the post-publish optimizer for a new channel

SKILL.md has a plain-English intro, a worked example, the exact commands and the known pitfalls.

## Responsible use

The automation must never auto-delete, auto-privatize or auto-swap packaging on a video. QA gates alert; humans decide. Only manage channels you own or are authorized to manage, and you are responsible for YouTube API terms and quota use. OpusClip, Repurpose.io and VidIQ are trademarks of their owners; this skill is not affiliated with or endorsed by them.

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

## Zero approval loop vs never auto-swap packaging

These two ideas fit together because they cover different things:

- **No approval loop (metadata):** tags, category, recording date and standing defaults (license, madeForKids, embeddable) are applied automatically each night, and every write is verified by read-back.
- **Always a human decision (packaging):** titles and thumbnails are never changed by the automation. After 72 hours it may propose a change, and a human approves it. Titles and descriptions are written at schedule time and the optimizer never edits them (outside the optional sponsor-block flow).
- **QA gates only alert:** audio verification and the day 7 retention and CTR review never delete, privatize or change anything.

## One-time setup note

Creating the Google Cloud project, enabling the API and approving the OAuth consent screen are done once by a person in a browser. The automation itself follows the spec's NO BROWSER rule: if it ever seems to need a browser, it reports that and does not click.

## License

MIT, see LICENSE.
