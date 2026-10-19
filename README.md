# DailySiteDigest

Reads a list of RSS/Atom feeds once a day and emails an HTML digest of new items, with images and video thumbnails, through Outlook desktop.

> **Community plan only.** The project depends on `UiPath.System.Activities` and `UiPath.Mail.Activities` from the Studio installer feed, and nothing else. It uses no Integration Service, Microsoft 365 connector, GenAI, agent, Healing Agent, ScreenPlay or serverless robot, so it spends no Community credits. Fetching, parsing and HTML building run in `Invoke Code` (VB.NET, `HttpClient` + `System.Xml.Linq`).

## Architecture

| Workflow | Purpose |
|----------|---------|
| `Main.xaml` | Prepare data folder, run the steps, mark reported links as seen |
| `LoadConfig.xaml` | Read the three Orchestrator assets |
| `FetchFeeds.xaml` | GET each enabled feed, parse items and media, drop seen/old, rank |
| `BuildDigestHtml.xaml` | Build the Outlook-safe HTML email and save an archive copy |
| `SendDigest.xaml` | `Send Outlook Mail Message` with `IsBodyHtml=True` |

"Notable" is a fixed rule: per site, items with an image or video come first, then the newest, up to `MaxItemsPerSite`. Images come from `media:thumbnail`, `media:content`, an image `enclosure`, or the first `<img>` in the item HTML. YouTube items (`yt:videoId`) and video enclosures are labelled VIDEO and link to the video page, because email clients cannot play video inline.

## Prerequisites

### Orchestrator assets (in the folder the process runs in)

| Asset | Type | Value |
|-------|------|-------|
| `DigestRecipient` | Text | Email address to send the digest to |
| `LookbackDays` | Integer | `2`: ignore items older than this (values below 1 fall back to 2) |
| `MaxItemsPerSite` | Integer | `5`: per-site cap (values below 1 fall back to 5) |

There are no secrets. Outlook sends as the signed-in user.

### Robot machine

- Outlook desktop (classic) installed and signed in for the robot user. The "new Outlook" app does not support the Outlook desktop activities.
- Outbound HTTPS to the feed hosts.
- The single Community Unattended Robot, or attended runs from Studio.

## Data folder

Runtime state is kept outside the package so it survives republishing:

`%LOCALAPPDATA%\DailySiteDigest\`

| File | Purpose |
|------|---------|
| `Sites.csv` | `SiteName,FeedUrl,Enabled`, copied from `Data\Sites.csv` on the first run, then yours to edit |
| `Seen.txt` | One reported link per line. Appended only after the email sends, so a failed send is retried the next day |
| `Output\Digest_yyyy-MM-dd.html` | Archive copy of each email |

Delete `Seen.txt` to report everything within the lookback window again.

## First open in Studio

1. Open `project.json` in Studio 25.10+ and let it restore packages.
2. Open `SendDigest.xaml` and confirm `Send Outlook Mail Message` shows no error. If it does, drop a fresh Send Outlook Mail Message activity in, set To / Subject / Body / IsBodyHtml as in the original, and delete the old one.
3. Create the three assets, then run `Main.xaml`.

## Scheduling

Publish to Orchestrator, create the process in your folder, and add a daily **Time trigger** that targets the Unattended Robot.

## Verifying a run

- Logs: `[Site Digest] <site>: N new item(s)`, `... new item(s) to report`, `Sent '<subject>' to <recipient>`.
- Inbox: subject `Daily site digest - d MMM yyyy (N new)`, or `(no new updates)`.
- A feed that fails is logged and listed in a red box at the bottom of the email. The rest of the run carries on.

## Extending config

- Add sites: add rows to `%LOCALAPPDATA%\DailySiteDigest\Sites.csv`. Set `Enabled` to `FALSE` to pause a site. Many sites expose `/feed`, `/rss` or `/rss.xml`. For a YouTube channel, use `https://www.youtube.com/feeds/videos.xml?channel_id=<id>`.
- Pages without RSS/Atom are not supported. Scraping them needs browser automation and breaks when the page layout changes.

## Not yet implemented

- Images are linked, not embedded, so Outlook may ask to download pictures until you trust the sender.
- No AI summarisation, because GenAI activities spend Community credits.
