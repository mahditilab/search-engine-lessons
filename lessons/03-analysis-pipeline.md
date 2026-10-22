# Lesson 3: The analysis pipeline

*2026-10-22 · [All lessons](../README.md)*

## Why it matters
Lesson 2 said the term dictionary holds "what the analyzer produced". Now we look at that analyzer. It decides which terms exist, so it decides what can be found at all: if the analyzer turns `Brake-Pads` into `brake` and `pads`, a search for `pads` matches; if it keeps `brake-pads` as one term, it doesn't. Most "why doesn't my search find this?" problems are analysis problems.

## Core explanation

An **analyzer** turns a string into a list of **tokens** (which become terms). It is always a chain of exactly three kinds of components, in this order:

1. **Character filters** (zero or more) work on the raw string *before* it is split. Example: strip HTML tags, replace `&` with `and`. (Lesson 4.)
2. **Tokenizer** (exactly one) splits the string into tokens and records each token's **position** and character **offsets**. Example: the `standard` tokenizer splits on word boundaries following the Unicode text segmentation rules. (Lesson 5.)
3. **Token filters** (zero or more) transform, drop or add tokens *after* splitting. Example: lowercase, remove stop words, stem, add synonyms. (Lessons 6–9.)

```
"<p>Brake Pads for VW-Golf 7</p>"
   │  char filter:   html_strip
   ▼
"Brake Pads for VW-Golf 7"
   │  tokenizer:     standard
   ▼
[Brake] [Pads] [for] [VW] [Golf] [7]
   │  token filter:  lowercase
   ▼
[brake] [pads] [for] [vw] [golf] [7]
   │  token filter:  stop (english)
   ▼
[brake] [pads] [vw] [golf] [7]        ← these go into the term dictionary
```

**Order matters.** Filters run in the order you list them. A stop-word filter that expects lowercase words must come *after* the lowercase filter, otherwise `The` slips through. Char filters only ever see text, token filters only ever see tokens.

**Positions survive filtering.** When a token is removed, the tokens after it keep their original positions, so there is a gap. That is what lets phrase queries (lesson 17) still respect the original word distances.

**Built-in analyzers.** You rarely start from scratch. `standard` is the default for `text` fields: standard tokenizer + lowercase filter (no stop words in modern versions). Others include `simple` (split on non-letters, lowercase), `whitespace`, `keyword` (the entire input as one token, no analysis) and language analyzers such as `english` or `german`. When none fits, you define a **custom analyzer** by naming a tokenizer plus filter lists.

**Same chain, two moments.** The analyzer runs when a document is indexed (producing the terms stored in the index) and again on the query text of a `match` query (producing the terms to look up). If both produce the same terms, they meet in the dictionary. Lesson 11 covers how and why you may choose different analyzers for each side.

**Analysis only affects `text` fields.** `keyword` fields skip the chain and index the value as a single exact term (lesson 12). A `term` query is also not analyzed: it looks up exactly what you give it.

## Concrete example

Input: `<p>Brake Pads for VW-Golf 7 &amp; Disc</p>`

| Stage | Result |
|---|---|
| after `html_strip` | `Brake Pads for VW-Golf 7 & Disc` (tags removed, `&amp;` decoded to `&`) |
| after `standard` tokenizer | `Brake`, `Pads`, `for`, `VW`, `Golf`, `7`, `Disc` (the hyphen splits `VW-Golf`; the lone `&` is dropped) |
| after `lowercase` | `brake`, `pads`, `for`, `vw`, `golf`, `7`, `disc` |
| after `stop` | `brake`, `pads`, `vw`, `golf`, `7`, `disc` |

Final positions: `brake`=0, `pads`=1, (`for`=2 removed, gap), `vw`=3, `golf`=4, `7`=5, `disc`=6.

You can run exactly this chain ad hoc with the `_analyze` API. Add `"explain": true` to see the output of every stage:

```http
POST /_analyze
{
  "char_filter": ["html_strip"],
  "tokenizer": "standard",
  "filter": ["lowercase", "stop"],
  "text": "<p>Brake Pads for VW-Golf 7 &amp; Disc</p>",
  "explain": true
}
```

And as a reusable custom analyzer on an index:

```http
PUT /parts-lesson3
{
  "settings": {
    "analysis": {
      "analyzer": {
        "parts_text": {
          "type": "custom",
          "char_filter": ["html_strip"],
          "tokenizer": "standard",
          "filter": ["lowercase", "stop"]
        }
      }
    }
  },
  "mappings": {
    "properties": { "name": { "type": "text", "analyzer": "parts_text" } }
  }
}

POST /parts-lesson3/_analyze
{ "analyzer": "parts_text", "text": "<p>Brake Pads for VW-Golf 7 &amp; Disc</p>" }

DELETE /parts-lesson3
```

## Common pitfalls
- **Assuming a search matches the stored text.** It matches the *terms*. `_source` still shows the original `VW-Golf`, but the index only knows `vw` and `golf`.
- **Wrong filter order.** `stop` before `lowercase` leaves capitalised stop words in. Always test the chain with `_analyze`.
- **Changing an analyzer on an existing field.** Terms already in segments were built with the old chain. You must reindex; you cannot update the analyzer of an existing field in place.
- **Querying a `keyword` field with analyzed text.** `term` on a `keyword` field for `brake pads` needs the exact stored value, including case, because no analysis happens on either side.

## Quick quiz
1. In which order do the three kinds of analysis components run, and how many of each can a custom analyzer have?
2. For the input `The Oil Filter`, what are the tokens with `["stop", "lowercase"]` versus `["lowercase", "stop"]` (English stop words)?
3. A `match` query for `VW-Golf` finds a document stored as `vw golf`. Why?

## Tomorrow
Lesson 4: **character filters**, `html_strip`, `mapping` and `pattern_replace`, the first stage of the chain.

---

<details>
<summary><strong>Answers</strong></summary>

1. Character filters → tokenizer → token filters. Zero or more char filters, exactly one tokenizer, zero or more token filters.
2. `["stop", "lowercase"]`: `the`'s capitalised form `The` is not in the (lowercase) stop list at that point, so the result is `the`, `oil`, `filter`. `["lowercase", "stop"]`: `oil`, `filter`. (Note: the `stop` filter's `_english_` list matches case-sensitively unless you lowercase first.)
3. The query text goes through the same analyzer: `VW-Golf` becomes `vw`, `golf`, the same terms the document produced at index time, so the postings lists match.

</details>
