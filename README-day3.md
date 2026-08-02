# CSV Processing Pipeline (n8n)

Downloads a CSV over HTTP, parses it into items, enriches each row with derived fields, computes grouped statistics across the whole dataset, writes the result back out to a file, and emails it as an attachment.

Built to learn the Code node's two execution modes and how binary data moves through n8n — the skeleton underneath every document-processing automation.

---

## Dataset

Titanic passenger manifest — 891 records, 12 columns.

```
https://raw.githubusercontent.com/datasciencedojo/datasets/master/titanic.csv
```

Chosen because it has real-world flaws: `Age` is missing on 177 rows, everything arrives as strings, and `Survived` is `0`/`1` rather than a boolean. Clean sample data teaches nothing.

---

## Architecture

```
Manual Trigger
     ↓
HTTP Request              Response Format: File → binary          1 item
     ↓
Extract from File         CSV → items                           891 items
     ↓
Code: Enrich Rows         run once for EACH item                891 items
     ↓                    parse types, derive ageBracket,
     ↓                    title, classLabel, familySize
     ↓
Code: Survival by Class   run once for ALL items                  3 items
     ↓                    group by class, compute rates
     ↓
Convert to File           items → CSV binary                      1 item
     ↓
Send Email                with attachment
```

Note the item counts: `1 → 891 → 891 → 3 → 1`. Binary to structured data and back again.

---

## Output

| class | passengers | survivors | survivalRate | avgFare | avgAge | ageDataMissing |
|---|---|---|---|---|---|---|
| First | 216 | 136 | 63.0% | 84.15 | 38.2 | 30 |
| Second | 184 | 87 | 47.3% | 20.66 | 29.9 | 11 |
| Third | 491 | 119 | 24.2% | 13.68 | 25.1 | 136 |

`ageDataMissing` is reported alongside `avgAge` deliberately — an average computed from 355 of 491 rows should say so.

---

## What I learned

**Items carry two kinds of payload.** Alongside `json`, an item can hold a `binary` key with a named property containing `mimeType`, `fileName`, and content. Files aren't a separate track through the workflow; they're another key on the item you already have.

**Response Format decides everything.** Setting the HTTP Request node to `File` instead of `JSON` is what makes a CSV arrive as binary rather than being mangled into an unparseable string. This one dropdown is the difference between a working file pipeline and a confusing failure.

**The Code node's two modes are not interchangeable.**

*Run Once for Each Item* gives you `$json` for the current row and expects a single returned object. Right for per-row transformation.

*Run Once for All Items* gives you `$input.all()` — the entire array — and expects an array back. Right for anything needing to see the whole set: grouping, cross-row comparison, custom deduplication. Computing a survival rate per class is impossible per-item, because you can't know the group total until you've seen every row.

**The `json` wrapper is mandatory.** Every return, in both modes, must be `{ json: {...} }` or an array of them. Returning a bare `[{ class: 'First' }]` throws. This is the most common Code node error and worth making once deliberately.

**CSV gives you strings, always.** Every numeric field needs explicit conversion. Skipping it produces the same class of bug as a mistyped Set field — comparisons silently fail downstream.

**`Number('')` is `0`, not `null`.** A naive parse of the 177 blank Age values would claim those passengers were newborns and quietly poison every average in the output. The workflow would run green and the analysis would be wrong. Defensive parsing at the point of ingestion is the difference between an analysis and a wrong analysis.

**Field naming in the Set/Code step becomes the CSV headers.** Convert to File uses the incoming JSON keys directly, so the renaming done during enrichment is what gives the output clean lowercase headers instead of the source's inconsistent ones.

---

## Nodes introduced

`Code` (both modes) · `Extract from File` · `Convert to File` · HTTP Request Response Format · Send Email attachments

---

## Stack

n8n Cloud · JavaScript · SMTP
