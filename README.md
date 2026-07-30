# Hacker News AI Daily Digest 📰🤖

Fetches AI-related stories from Hacker News every morning, filters out low-signal posts, formats the top 5 into a readable digest, and emails it at 9:00 AM. Runs unattended on a schedule.

*Built as a first end-to-end n8n workflow to learn the platform's data model properly rather than following a tutorial.*

---

## 🚀 Overview

| Feature | Details |
| :--- | :--- |
| **Trigger** | Schedule — daily at 09:00 |
| **Source** | HN Algolia Search API (public, no auth) |
| **Output** | Formatted email digest of the top 5 stories |
| **Runtime** | ~3–6 seconds per execution |
| **Stack** | n8n Cloud · HN Algolia Search API · SMTP |

---

## 🧠 What I learned
An array inside an item is not multiple items: The API returns a single item containing a hits array of 30 objects. Downstream nodes see one item until Split Out explicitly expands it. This is the single biggest source of confusion in n8n and it's visible in the item counts on every connection.

Nodes run once per item automatically: No loop node is needed to iterate — loops exist for batching and rate-limiting, not iteration.

Line breaks in JSON: \n in a JSON view is a working newline, not a bug. JSON escapes line breaks for display. The expression editor's Result preview renders the real thing, and that preview (with its per-item stepper) is the most useful debugging surface in the tool.

Test your filters: A condition you've never seen reject anything is untested. The filter initially passed all 30 items because HN's relevance search already returns high-scoring stories. Raising the threshold until items were discarded proved it actually worked.

Data hygiene: Reshape data before it leaves the pipeline. The raw API response carries ~15 fields per story including _highlightResult markup and Algolia metadata. Reducing to 4 chosen fields matters as soon as the next step is a database write or an LLM prompt, where junk fields become cost and noise.


## 🏗️ Architecture

The item counts are the interesting part. The workflow deliberately goes one-to-many (`Split Out`) and then many-to-one (`Aggregate`), because that round trip is the core of how n8n moves data.

```text
Schedule Trigger
      ↓
HTTP Request        GET [hn.algolia.com/api/v1/search](https://hn.algolia.com/api/v1/search)   → 1 item (30 stories nested)
      ↓
Split Out           field: hits                        → 30 items
      ↓
Filter              points > 2000                      → 11 kept / 19 discarded
      ↓
Clean Story Data    keep title, url, points, author    → 11 items, 4 fields each
      ↓
Sort                points, descending                 → 11 items
      ↓
Limit               max 5, keep first                  → 5 items
      ↓
Aggregate           all item data → field: data        → 1 item
      ↓
Build Message       compose digest string              → 1 item
      ↓
Send an Email       SMTP                               → delivered
