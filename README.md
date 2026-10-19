<p align="center"><a href="https://apify.com/DevilScrapes/socialblade-stats-scraper"><img src="assets/banner.svg" alt="SocialBlade Stats Scraper — JavaScript example" width="100%"></a></p>

# SocialBlade Scraper for Node.js: account stats as JSON for your backend

<p align="center">
<a href="https://apify.com/DevilScrapes/socialblade-stats-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="JavaScript" src="https://img.shields.io/badge/Node.js-20%2B-339933?logo=nodedotjs&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/socialblade-scraper-javascript/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/socialblade-scraper-javascript/actions/workflows/ci.yml/badge.svg"></a>
</p>

This repository calls the **[SocialBlade Stats Scraper](https://apify.com/DevilScrapes/socialblade-stats-scraper)** from Node.js through the official `apify-client` package. Your service passes in handles or SocialBlade profile URLs for YouTube, Twitch, Instagram and TikTok; Apify does the fetching and returns plain objects with audience counts, SocialBlade's grade, ranks and an optional array of daily snapshots. There is no build step and no extra dependency beyond the client, so the core call drops into an Express route, a scheduled worker or a cloud function.

> **Actor:** [SocialBlade Stats Scraper](https://apify.com/DevilScrapes/socialblade-stats-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in Python:** [socialblade-scraper-python](https://github.com/devilscrapes/socialblade-scraper-python)

## 🔥 What you get

- **Four networks, one schema**: a YouTube channel and a TikTok account land in the same columns, with `follower_count` meaning subscribers on YouTube and followers everywhere else.
- **Name a target two ways**: a `platform` plus `handle` pair, or a full SocialBlade profile URL, which wins if both are present. Mix networks freely in one `targets` list.
- **Daily history on request**: with `includeHistory` on, `growth_history` holds up to 15 dated snapshots of followers, plus views and content count where the network has them. Switch it off for lighter rows.
- **Network-specific extras stay null elsewhere**: Instagram rows add engagement rate and average likes and comments, TikTok rows add lifetime likes and the verified flag, YouTube rows add total views.
- **Counts are numbers**: figures SocialBlade delivers as text are converted to integers, so sorting and arithmetic work without cleanup.
- **One dead handle does not end the run**: throttled or failed requests are retried up to five times with growing pauses on a new proxy session, and a target that still fails is logged and skipped.

## 🎯 Use cases

### Show audience stats inside your product

A brand dashboard or a media kit generator can fetch numbers for a company's official accounts on demand and render them next to each other. The shared field names mean one front-end component covers every network.

### Alert when an account's growth stalls

Run `node index.js` daily from a container or a scheduled function, compare each account's latest daily gain with its average, and post a warning to Slack when the gap gets large. The recipe below already does the comparison.

### Enrich CRM records for partnership teams

Store the follower count, grade and `source_url` on each partner record in your CRM. Refreshing them weekly through a queue job keeps account managers from quoting stale audience sizes in proposals.

### Process long target lists with webhooks

When a list is too long to wait for inside a request, start the run with `start()`, attach an Apify webhook for the succeeded event and read the dataset from the endpoint the webhook calls. Your HTTP handler returns immediately.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/socialblade-scraper-javascript.git
cd socialblade-scraper-javascript
npm install
export APIFY_TOKEN=your_token_here
node index.js                       # uses input.json
node index.js my-input.json --out data --max-charge 2
```

The script starts the [SocialBlade Stats Scraper](https://apify.com/DevilScrapes/socialblade-stats-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

These four statements are the whole integration. `call()` waits for the run to end, then `listItems()` returns the rows from its default dataset.

```javascript
import { ApifyClient } from 'apify-client';

const client = new ApifyClient({ token: 'YOUR_APIFY_TOKEN' });
const run = await client.actor('DevilScrapes/socialblade-stats-scraper').call({"targets": [{"platform": "youtube", "handle": "mrbeast6000"}], "includeHistory": true, "maxItems": 1, "proxyConfiguration": {"useApifyProxy": true, "apifyProxyGroups": ["RESIDENTIAL"]}});
const { items } = await client.dataset(run.defaultDatasetId).listItems();
console.log(items);
```

## 📥 Input

`index.js` submits `input.json` as is. It names one large public YouTube channel, the same target the daily platform test checks, and caps the run at a single result with daily history included. Each entry in `targets` is an object; build the array from your own database rows to track whatever accounts your service follows.

```json
{
  "targets": [
    {
      "platform": "youtube",
      "handle": "mrbeast6000"
    }
  ],
  "includeHistory": true,
  "maxItems": 1,
  "proxyConfiguration": {
    "useApifyProxy": true,
    "apifyProxyGroups": [
      "RESIDENTIAL"
    ]
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `targets` | array | yes | `[{"platform": "youtube", "handle": "mrbeast6000"}]` | — |
| `includeHistory` | boolean | no | `true` | — |
| `maxItems` | integer | no | `100` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true, "apifyProxyGroups": ["RESIDENTIAL"]}` | — |

## 📤 Output

Every object describes one account on one network. Count fields are integers or `null`, `scraped_at` is an ISO 8601 string in UTC, and `growth_history` dates are ISO strings too, so they sort correctly as text. Fields specific to another network come back as `null`, so guard for them in code that handles mixed platforms.

| Field | Type | Always present |
|---|---|---|
| `platform` | string | yes |
| `handle` | string | yes |
| `channel_id` | string or null | no |
| `display_name` | string or null | no |
| `avatar_url` | string or null | no |
| `follower_count` | integer | yes |
| `view_count` | integer or null | no |
| `content_count` | integer or null | no |
| `grade` | string | yes |
| `country` | string or null | no |
| `ranks` | object or null | no |
| `engagement_rate` | number or null | no |
| `avg_likes` | number or null | no |
| `avg_comments` | number or null | no |
| `total_likes` | integer or null | no |
| `verified` | boolean or null | no |
| `growth_history` | array | yes |
| `source_url` | string | yes |
| `scraped_at` | string | yes |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result emitted | $0.001 |

Prices as configured on 2026-10-16. The [Actor page](https://apify.com/DevilScrapes/socialblade-stats-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in JavaScript

This script derives day-over-day follower gains from each row's history, compares the most recent day with the account's average and flags anything that has dropped below half its usual pace. It uses Node.js built-ins only.

```javascript
import { readFile } from 'node:fs/promises';

const rows = JSON.parse(await readFile('results.json', 'utf8'));

const report = rows.map((row) => {
    const days = [...(row.growth_history ?? [])]
        .filter((d) => d.follower_count != null)
        .sort((a, b) => a.date.localeCompare(b.date));
    const gains = days.slice(1).map((d, i) => d.follower_count - days[i].follower_count);
    const average = gains.length ? gains.reduce((sum, g) => sum + g, 0) / gains.length : null;
    const latest = gains.at(-1) ?? null;
    return {
        account: `${row.platform}/${row.handle}`,
        followers: row.follower_count,
        grade: row.grade,
        avgDailyGain: average === null ? null : Math.round(average),
        latestGain: latest,
        slowing: average !== null && latest !== null && latest < average * 0.5,
    };
});

console.table(report);
const flagged = report.filter((r) => r.slowing).map((r) => r.account);
if (flagged.length) console.log(`Growth below half the usual pace: ${flagged.join(', ')}`);
```

## ❓ FAQ

### Is this an official SocialBlade API?

No. It reads the figures SocialBlade's public profile pages display and returns them as JSON, and it has no connection to SocialBlade the company. Your service needs only an Apify token.

### Is it usable from TypeScript?

Yes. `apify-client` bundles its type definitions. Write an interface for the row, with `growth_history` as an array of dated snapshots, and cast `listItems()` results to it for autocompletion.

### What does a run cost when nothing comes back?

The $0.20 start event is charged once a valid input starts processing, even if every target then fails. A run with no rows ends as failed, so check `run.status` before trusting an empty array. Invalid input is rejected before the start charge.

### Will duplicate entries in `targets` be merged?

No. Each entry is fetched and charged on its own, so remove repeated handles in your code before you call the Actor.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/socialblade-stats-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [SocialBlade Stats Scraper on Apify Store](https://apify.com/DevilScrapes/socialblade-stats-scraper)
- [Python version of this example](https://github.com/devilscrapes/socialblade-scraper-python)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [YouTube Channel Videos Scraper](https://apify.com/DevilScrapes/youtube-channel-scraper) — channel metadata and recent videos straight from YouTube.
- [YouTube Shorts Sponsorship Signals Scraper](https://apify.com/DevilScrapes/youtube-shorts-sponsorship-signals) — spot sponsored Shorts before you pitch a channel.
- [Patreon Creator Scraper](https://apify.com/DevilScrapes/patreon-creator-scraper) — patron counts and tiers for the creator side of the economy.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/socialblade-stats-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with SocialBlade.
