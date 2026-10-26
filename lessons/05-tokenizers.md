# Lesson 5: Tokenizers

*2026-10-26 · [All lessons](../README.md)*

## Why it matters
The tokenizer decides what a "word" is, and every later step (filters, stemming, scoring) only sees what it emits. A wrong tokenizer cannot be repaired downstream: if `0986-494` is split in two, no filter will glue it back together. For part numbers, model names and emails, this choice often decides whether a search works.

## Core explanation

In the pipeline from lesson 3, an analyzer has exactly **one tokenizer**. It receives the text left over from the char filters (lesson 4) and cuts it into a stream of **tokens**. It also records each token's position and character offsets, which phrase queries and highlighting use later.

The built-in tokenizers you will meet most often:

| Tokenizer | Rule | Typical use |
|---|---|---|
| `standard` | Unicode text segmentation (UAX #29): splits on whitespace and most punctuation, drops the punctuation | Default for `text` fields; good for natural language |
| `whitespace` | Splits on whitespace only, keeps punctuation and case | Pre-formatted codes where punctuation matters |
| `keyword` | Emits the **entire input as one token** | Normalizing an exact-match field (with filters, e.g. lowercase) |
| `letter` | Splits at every non-letter character, so digits are dropped | Rarely right; fine for pure-text data |
| `pattern` | Splits on a regex (default `\W+`) | Custom separators, e.g. `[,;]` |
| `uax_url_email` | Like `standard`, but keeps URLs and email addresses as single tokens | Text with emails or links |

Things worth knowing:

- **Tokenizers don't lowercase.** The `standard` *tokenizer* keeps case. The `standard` *analyzer* is that tokenizer plus a `lowercase` token filter (lesson 6). The `pattern` *analyzer* does lowercase by default, but the `pattern` tokenizer does not.
- **`standard` is smart, not simple.** It splits `Golf-7` into `Golf` and `7`, but it keeps `Pad_X1` together (underscore joins) and `3.5` together (a dot between digits stays).
- **`keyword` is not the `keyword` field type.** The field type stores the raw value as a single term. The `keyword` tokenizer does the same cutting-nothing, but inside an analyzer, so token filters can still run afterwards.
- `standard` has a `max_token_length` setting (default 255); longer tokens are split at that length.

## Concrete example

Input text: `Brake-Pad_X1, VW Golf-7 0986-494 sales@example.com`

| Tokenizer | Tokens |
|---|---|
| `standard` | `Brake` · `Pad_X1` · `VW` · `Golf` · `7` · `0986` · `494` · `sales` · `example.com` |
| `whitespace` | `Brake-Pad_X1,` · `VW` · `Golf-7` · `0986-494` · `sales@example.com` |
| `keyword` | `Brake-Pad_X1, VW Golf-7 0986-494 sales@example.com` (one token) |
| `letter` | `Brake` · `Pad` · `X` · `VW` · `Golf` · `sales` · `example` · `com` |
| `uax_url_email` | `Brake` · `Pad_X1` · `VW` · `Golf` · `7` · `0986` · `494` · `sales@example.com` |

Look at what each one does to the part number `0986-494`: `standard` splits it into two tokens, `whitespace` keeps it intact, and `letter` throws it away entirely.

Try it (no index needed):

```http
GET /_analyze
{
  "tokenizer": "standard",
  "text": "Brake-Pad_X1, VW Golf-7 0986-494 sales@example.com"
}

GET /_analyze
{
  "tokenizer": "whitespace",
  "text": "Brake-Pad_X1, VW Golf-7 0986-494 sales@example.com"
}

GET /_analyze
{
  "tokenizer": "uax_url_email",
  "text": "Brake-Pad_X1, VW Golf-7 0986-494 sales@example.com"
}

# A custom pattern tokenizer inside an analyzer, split on ; or ,
PUT /parts-lesson5
{
  "settings": {
    "analysis": {
      "tokenizer": {
        "semi_comma": { "type": "pattern", "pattern": "[;,]\\s*" }
      },
      "analyzer": {
        "list_analyzer": { "type": "custom", "tokenizer": "semi_comma" }
      }
    }
  }
}

GET /parts-lesson5/_analyze
{ "analyzer": "list_analyzer", "text": "VW Golf; Audi A3, Seat Leon" }

DELETE /parts-lesson5
```

The last `_analyze` call returns `VW Golf`, `Audi A3`, `Seat Leon`.

## Common pitfalls
- **Assuming the tokenizer lowercases.** It doesn't. Add a `lowercase` token filter, or you will see `Brake` in the index and wonder why `brake` misses.
- **Letting `standard` split identifiers.** `0986-494` becomes `0986` and `494`, so a query for just `494` hits it. If part numbers must match as a unit, index them as `keyword` or use a dedicated analyzer.
- **Using `letter` on data with digits.** `Golf7` becomes `Golf`, and numeric parts vanish from the index.
- **Picking `keyword` for a `text` field.** The whole value is one token, so `match` on one word of a sentence finds nothing.

## Quick quiz
1. What tokens does the `standard` tokenizer produce for `Golf-7 TDI`, and which of them would a search for `golf` match, assuming no lowercase filter?
2. You want `sales@example.com` to stay one token inside a description field. Which tokenizer?
3. What is the difference between the `keyword` field type and the `keyword` tokenizer?

## Tomorrow
Lesson 6: **token filters** (lowercase, ASCII folding, stop words), the step that cleans up what the tokenizer emits.

---

<details>
<summary><strong>Answers</strong></summary>

1. `Golf`, `7`, `TDI`. With no lowercase filter, a query for `golf` does not match `Golf`, because the tokenizer preserves case and the terms differ. (The standard *analyzer* would match, since it adds lowercase.)
2. `uax_url_email`. The `standard` tokenizer would split it into `sales` and `example.com`.
3. The field type stores the raw value as one term and skips analysis entirely. The tokenizer emits the whole input as one token inside an analyzer, so token filters (for example `lowercase`) can still process it afterwards.

</details>
