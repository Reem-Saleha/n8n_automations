# Multi-Source AI Feed (n8n)

Pulls AI stories from Hacker News across multiple pages and new papers from arXiv, normalises both into a common shape, deduplicates, then routes items into categories that each get handled differently before being combined into a single digest.

Built to learn branching, merging, and pagination — the three things that separate a linear workflow from a real one.

---

## Sources

| Source | Endpoint | Format | Auth |
|---|---|---|---|
| Hacker News | `hn.algolia.com/api/v1/search` | JSON | None |
| arXiv (cs.AI) | `export.arxiv.org/api/query` | XML | None |

---

## Architecture

```
Manual Trigger
   │
   ├──→ HTTP Request (HN, paginated ×3)  → 3 items (one per page)
   │       ↓ Split Out: hits              → 150 items
   │       ↓ Edit Fields (normalise)      → 150 items, source = "HN"
   │       │
   │       └──────────────┐
   │                      ↓
   └──→ HTTP Request (arXiv)         Merge (Append)  → 180 items
           ↓ XML → JSON                   ↓
           ↓ Split Out: feed.entry    Remove Duplicates (on id)
           ↓ Edit Fields (normalise)      ↓
           │  source = "arXiv"         Switch (Rules)
           └──────────────┘            ├─→ Research  (source = arXiv)   → 30
                                       ├─→ Hot       (score > 1000)     → 76
                                       └─→ Fallback  (everything else)  → 74
```

Each Switch branch then runs its own `Sort → Limit → Aggregate → Set` chain, and the branches merge back into one email body before delivery.

### Pagination config

On the HN HTTP Request node, under Options → Pagination:

```
Mode:              Update a Parameter in Each Request
Type:              Query
Name:              page
Value:             {{ $pageCount }}
Complete When:     Other
Complete Expr:     {{ $pageCount >= 3 }}
```

`$pageCount` is zero-indexed and only exists inside pagination settings. Pagination returns **one item per page**, not a flattened list — a `Split Out` is still required afterward.

### Normalised schema

Both sources are mapped to the same five fields before merging:

| Field | Type | HN source | arXiv source |
|---|---|---|---|
| `title` | String | `title` | `title` |
| `url` | String | `url` | `id` (abstract link) |
| `score` | Number | `points` | `0` (no equivalent) |
| `source` | String | `"HN"` (fixed) | `"arXiv"` (fixed) |
| `id` | String | `objectID` | `id` |


---

## What I learned

**Expressions are stored with a leading `=`.** This was the root cause of nearly every bug in this build. Internally n8n stores `{{ $json.url }}` as literal text and `={{ $json.url }}` as an evaluated expression; the Fixed/Expression toggle just adds or removes that character. Four fields were silently holding the literal string `{{ $json.url }}` instead of data.

It masqueraded as a *type* error — a field containing the literal text `{{ $json.points }}` is a String, so the Switch's numeric comparison failed with "wrong type," and enabling "Convert types where required" appeared to fix it. Several rounds of debugging went into the symptom before finding the cause one level down.

**The Result preview is ground truth.** Under any expression field, it shows the evaluated value with a per-item stepper. If it echoes `{{ $json.url }}` back at you, the `=` is missing. Fifteen seconds of checking would have caught all four.

**A fallback is not a rule.** Writing a Routing Rule comparing `fallback` to `value2` compares two literal strings and matches nothing, ever. The real mechanism is the Fallback Output dropdown set to "Extra Output," which adds a connector catching everything unmatched.

**Filter vs IF vs Switch.** Two outcomes → IF. Three or more → Switch. Building a Filter and then a second Filter with the inverted condition is a bug factory — change one threshold and you'll forget the other.

**Merge modes do completely different things.** Append stacks (5 + 3 = 8). Combine joins on a matching field like SQL. Combine-by-position pairs blindly by index and is almost never what you want. Merge also waits for all inputs by default, so an empty branch can stall it in a way that looks like a hang rather than an error.

**Normalise at the edge, stamp the source at ingestion.** Mapping `points` → `score` inside each source's own Set node means downstream logic never has to know which API a row came from. Adding a fixed `source` field means you still can tell when you need to.

**Item counts on connections are the primary diagnostic.** The arXiv branch produced 11 items when it should have produced 30 — a Split Out on `feed` instead of `feed.entry`. The number was visible on the canvas the whole time. A surprising count is a signal to open the JSON, not a detail to move past.

---

## Nodes introduced

`Switch` · `Merge` · `Remove Duplicates` · `XML` · HTTP Request pagination options

---

## Stack

n8n Cloud · HN Algolia API · arXiv API · SMTP
