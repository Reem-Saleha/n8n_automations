# n8n Automations

Learning n8n properly — workflow JSON, architecture notes, and what broke along the way.

Each project folder contains an importable `workflow.json`, a canvas screenshot, and a README covering how it's built, what I learned, and what I'd do differently. The last two sections are the point; the workflows themselves are small.

---

## Projects

| # | Project | What it does | Key concepts |
|---|---|---|---|
| 01 | [AI News Digest](./01-ai-news-digest) | Emails a daily digest of top AI stories from Hacker News | The item model, Split Out / Aggregate, scheduling, SMTP |
| 02 | [Multi-Source AI Feed](./02-multi-source-feed) | Merges paginated HN results with arXiv papers, dedupes, routes by category | Pagination, Switch routing, Merge modes, schema normalisation |
| 03 | [CSV Processing Pipeline](./03-csv-pipeline) | Downloads a CSV, enriches rows, computes grouped stats, emails the result as a file | Code node (both modes), binary data, file conversion |

---

## Running any of these

1. Open n8n → **Workflows → Import from File** and select the project's `workflow.json`. Alternatively, copy the file contents and paste directly onto the canvas with `Ctrl+V` — a workflow is just JSON underneath.
2. Create whatever credentials that project's README lists. Credentials are stored separately in n8n and are **not** included in these exports, so no secrets travel with the files.
3. Run manually first, then publish if it has a schedule trigger.

Everything here uses free, public, unauthenticated APIs. The only credential any project needs is SMTP for email delivery.

---

## Things that took me longest to understand

Collected across projects, since they came up more than once.

**An array inside an item is not multiple items.** An API returning 30 records in a `hits` array is *one* item until `Split Out` expands it. The item counts printed on every canvas connection are the primary diagnostic — a surprising number means open the JSON, not move on.

**Expressions are stored with a leading `=`.** `{{ $json.url }}` is literal text; `={{ $json.url }}` is evaluated. The Fixed/Expression toggle just adds or removes that character. When a field silently holds literal `{{ }}` text, the resulting failure shows up downstream as a *type* error, which sends you debugging the wrong thing entirely.

**Fix data types where data is created, not where it's consumed.** Patching a mismatch at the comparison means the next node that touches the field breaks too. "Convert types where required" makes broken comparisons appear to work and hides real defects — leave it off.

**A condition you've never seen reject anything is untested.** Raise the threshold until items actually get discarded, then set it back.

**Green checkmarks mean the node didn't throw, not that the output is correct.** A workflow can run entirely clean and produce garbage — literal `{{ }}` strings in every field, a Split Out on the wrong path, a routing rule that can never match. Reading the data at each step is not optional.

---

## Stack

n8n Cloud · JavaScript expressions · REST + XML APIs · SMTP

---

**Reem** — BS Artificial Intelligence, NUST SEECS
