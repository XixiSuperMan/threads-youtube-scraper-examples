# YouTube & Threads monitoring — runnable examples

Small, dependency-light examples for two Apify Actors that watch a platform for **new** content and return only what appeared since your last run.

- **[YouTube Scraper | Monitor New Videos | No API Key](https://apify.com/reportable_broth/youtube-scraper-monitor)** — no Google Cloud project, no quota units
- **[Threads Scraper | No Login](https://apify.com/reportable_broth/threads-scraper-monitor)** — no account, no cookies

Pay per result, platform usage included: **$0.99 / 1,000 posts** on Threads and **$0.45 / 1,000 videos** on YouTube. With only-new mode on you are not charged twice for the same item.

Every snippet below was run against the live Actors before publishing.

---

## Setup

```bash
pip install "apify-client>=3"
export APIFY_TOKEN=your_token_here     # Settings -> API & Integrations
```

> **One gotcha up front.** On `apify-client` 3.x the object returned by `.call()` is a
> model, not a dict. `run["defaultDatasetId"]` raises
> `TypeError: 'Run' object is not subscriptable`. Use `run.default_dataset_id`
> (snake_case attribute). Older docs and blog posts still show the dict form.

---

## 1. Watch keywords on YouTube

```python
import os
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])

run = client.actor("reportable_broth/youtube-scraper-monitor").call(run_input={
    "keywords": ["ai agents", "web scraping"],
    "sort": "date",          # newest uploads first
    "onlyNew": True,         # only what you have not seen
    "maxAgeHours": 24,
})

for v in client.dataset(run.default_dataset_id).iterate_items():
    print(f"{v['published_text']:>14}  {v['channel']:<22}  {v['title']}")
    print(f"                {v['url']}")
```

The second run returns only genuinely new uploads. That is the point of the Actor.

Shorts come back too, flagged with `is_short` — for many keywords they are more than half of what YouTube returns. Set `"includeShorts": False` for long-form only, or raise `maxResults` to page past the first page.

## 2. Follow a set of channels

```python
run = client.actor("reportable_broth/youtube-scraper-monitor").call(run_input={
    "channels": ["@NASA", "@veritasium"],
    "onlyNew": True,
})
```

Handles work with or without `@`, and full channel URLs are accepted.

## 3. Watch a brand on Threads

```python
run = client.actor("reportable_broth/threads-scraper-monitor").call(run_input={
    "keywords": ["your brand", "competitor brand"],
    "modes": ["recent"],
    "onlyNew": True,
})

for p in client.dataset(run.default_dataset_id).iterate_items():
    print(f"@{p['user']['username']}  likes={p['like_count']}  {p['text'][:80]}")
```

## 4. Pull a Threads profile's history

```python
run = client.actor("reportable_broth/threads-scraper-monitor").call(run_input={
    "usernames": ["nike"],
    "maxPostsPerProfile": 300,
    "onlyNew": False,        # off for a one-off backfill
})
```

## 5. Find who is talking about a topic

```python
run = client.actor("reportable_broth/threads-scraper-monitor").call(run_input={
    "keywords": ["korean skincare"],
    "modes": ["recent", "top"],
    "findAccounts": True,
    "onlyNew": False,
})

accounts = [i for i in client.dataset(run.default_dataset_id).iterate_items()
            if i.get("_source") == "account"]

for a in sorted(accounts, key=lambda a: -a["posts_found"])[:10]:
    print(f"@{a['username']:<24} {a['posts_found']} posts  {a['total_likes']} likes")
```

Account records are aggregated from posts you already paid for, so they are not charged separately.

## 6. Send new items to Slack

```python
import os, requests
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])
SLACK_WEBHOOK = os.environ["SLACK_WEBHOOK"]

run = client.actor("reportable_broth/youtube-scraper-monitor").call(run_input={
    "keywords": ["your product name"],
    "sort": "date",
    "onlyNew": True,
})

for v in client.dataset(run.default_dataset_id).iterate_items():
    requests.post(SLACK_WEBHOOK, json={
        "text": f"*New video* - {v['channel']}\n{v['title']}\n{v['url']}"
    })
```

Put that on an Apify Schedule every 15 minutes and you have an alerting feed that never repeats itself.

---

## Notes

- **Timestamps on YouTube are approximate.** YouTube publishes relative text (`"12 days ago"`) on public pages, never an exact time. The field is called `published_ts_approx` for that reason — good enough for "in the last 24 hours", not good enough to sort uploads minutes apart.
- **An empty run is usually correct.** With `onlyNew` on, nothing new means nothing returned.
- **Residential proxy is the default** and worth keeping. Both platforms serve a stripped page to flagged IPs rather than returning an error, so a clean exit IP matters more than you would expect.
- Public content only. Neither Actor touches private, unlisted or members-only material.

## Licence

MIT for the examples in this repo. The Actors themselves are listed on Apify Store.
