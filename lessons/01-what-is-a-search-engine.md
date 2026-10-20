# Lesson 1: What a search engine is

*2026-10-20 · [All lessons](../README.md)*

## Why it matters
Every later topic (analyzers, BM25, shards, vectors) assumes you already think in the search engine's model: documents, fields, terms and relevance. Most bugs where search "doesn't find the obvious result" come from treating Elasticsearch like a SQL database. Getting this model right first saves a lot of debugging later.

## Core explanation

**Documents and fields.** In Elasticsearch/OpenSearch, a **document** is a JSON object, and it is the unit that gets stored and returned. Each top-level key is a **field**: `name`, `brand`, `price`, `part_number`. Every field has a type, set by the **mapping** (the index schema). The type controls how the field's value is stored and how it can be searched.

**An index is not a table.** An index is a set of documents, and it plays roughly the role a table plays in a database. The difference is what it's built for:

| | Relational database | Search engine |
|---|---|---|
| Built for | Correct transactions, joins, constraints | Finding the best matches fast across a lot of text |
| Answers | "Which rows match this predicate?" (yes/no) | "Which documents are most relevant, and in what order?" (score) |
| Text lookup | `LIKE '%brake%'` scans the rows | Looks the word up in a precomputed word → documents map |
| Consistency | Strong, ACID | Near-real-time: new docs become searchable after a *refresh*, about 1s by default |
| Joins | First-class | Mostly avoided; you denormalize instead |

In practice the database stays the **source of truth**, and the search index is a **derived, read-optimized copy** that you can rebuild from it.

**Full-text vs. exact match.** A string field can be indexed in two very different ways:

- **`text`**: the value goes through an **analyzer** (a pipeline that splits it into words and lowercases them, among other steps). The index stores the resulting **terms**. At query time, a full-text query (`match`) runs the *same* analysis on the query string and looks for documents that share terms. Results are scored by relevance. Use this for names, descriptions and anything a human types loosely.
- **`keyword`**: the value is stored **as one exact term**, without being split or lowercased. It can only be matched exactly, which also makes it usable for filtering, sorting and aggregations. Use this for IDs, SKUs, part numbers, brand codes, status enums.

Short version: a `text` field matches on the words in the value, and a `keyword` field matches only the whole value as stored.

## Concrete example

The document:
```json
{ "name": "Brake Pads for VW Golf 7", "part_number": "0 986 494 104" }
```

What's stored for each field:

| Field | Type | Stored terms |
|---|---|---|
| `name` | `text` (standard analyzer) | `brake`, `pads`, `for`, `vw`, `golf`, `7` |
| `part_number` | `keyword` | `0 986 494 104` (one term, spaces included) |

How four queries behave:

| Query | Matches? | Why |
|---|---|---|
| `match` name: `"vw brake"` | ✅ | The query becomes `vw`, `brake`, and both terms exist |
| `term` name: `"Brake"` | ❌ | `term` skips analysis, and the index holds `brake`, not `Brake` |
| `term` part_number: `"0 986 494 104"` | ✅ | Identical to the stored term |
| `term` part_number: `"0986494104"` | ❌ | Not the same string (no spaces) |

Try it (Kibana Dev Tools / OpenSearch Dashboards):

```http
PUT /parts-lesson1
{
  "mappings": {
    "properties": {
      "name":        { "type": "text" },
      "part_number": { "type": "keyword" }
    }
  }
}

PUT /parts-lesson1/_doc/1?refresh=true
{ "name": "Brake Pads for VW Golf 7", "part_number": "0 986 494 104" }

# See exactly what terms the text field produces
GET /parts-lesson1/_analyze
{ "field": "name", "text": "Brake Pads for VW Golf 7" }

# Full-text: hit, with a _score
GET /parts-lesson1/_search
{ "query": { "match": { "name": "vw brake" } } }

# Exact on text field: no hit (case mismatch)
GET /parts-lesson1/_search
{ "query": { "term": { "name": "Brake" } } }

# Exact on keyword: hit
GET /parts-lesson1/_search
{ "query": { "term": { "part_number": "0 986 494 104" } } }

DELETE /parts-lesson1
```

## Common pitfalls
- **Using `term` on a `text` field.** It looks for the raw string you passed, but the index holds analyzed, lowercased tokens. Use `match` on `text` and `term` on `keyword`.
- **Indexing part numbers as `text`.** The standard analyzer splits `"0 986 494 104"` into `0`, `986`, `494`, `104`, so a search for "494" matches unrelated parts. Identifiers should be `keyword`, plus an extra normalized field if you need flexible matching (lessons 4 and 12).
- **Treating the index as the source of truth.** It is near-real-time and gets rebuilt when mappings change, so keep the canonical data in your database.
- **Expecting a SQL-style yes/no answer.** A full-text query returns *ranked* results. Something can match and still rank 40th, and that is a relevance problem, not a missing-data problem.

## Quick quiz
1. You index `"status": "IN_STOCK"` as `text` and then run `term: { status: "IN_STOCK" }`. Why does it return nothing?
2. Which field type would you choose for (a) a product description, (b) an OE part number, (c) a brand you want to filter and facet on?
3. Name two reasons to keep a relational database as the source of truth even when all reads go through Elasticsearch.

## Tomorrow
Lesson 2: the **inverted index**, the term → postings-list structure that makes `match` fast and explains every result above.

---

<details>
<summary><strong>Answers</strong></summary>

1. The `text` field was analyzed, so the index holds the lowercased token `in_stock`. The standard tokenizer keeps underscores inside a word, so it stays one token. `term` doesn't analyze its input, so it looks for `IN_STOCK`, which isn't there. Map `status` as `keyword`.
2. (a) `text`. (b) `keyword`, possibly plus a normalized variant for flexible matching. (c) `keyword`, because filters, sorting and aggregations need the exact value.
3. Any two of these: ACID transactions and constraints; strong consistency, compared with search's refresh delay; the ability to rebuild or reindex search from it when mappings change; joins and relational integrity that the search index gives up by denormalizing.

</details>
