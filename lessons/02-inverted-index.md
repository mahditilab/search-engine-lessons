# Lesson 2: The inverted index

*2026-10-21 · [All lessons](../README.md)*

## Why it matters
Yesterday's `match` query found "Brake Pads for VW Golf 7" without scanning every document. The inverted index is what makes that possible, and it is the reason search engines are fast at "which documents contain X?" and slow at some other things. Once you can picture it, you can predict which queries are cheap (a term lookup), which are expensive (a leading wildcard), and why.

## Core explanation

**Forward vs. inverted.** A database row, or a document's `_source`, is a *forward* index: document → its words. To find every document containing `brake`, you would have to read them all. An **inverted index** flips this around: word → the documents that contain it. Every analyzed `text` field and every `keyword` field gets its own inverted index.

It has two main parts.

**1. The term dictionary.** This is the sorted list of every unique **term** in the field across the whole index (more precisely, per segment; segments come in lesson 20). A term is what the analyzer produced, so `Brake` became `brake` before it got here. Because the list is **sorted**, Lucene can find a term with a binary-search-like lookup, and it can also walk a range of terms cheaply. That is why `prefix` queries are reasonable and leading-wildcard queries (`*pads`) are not: a prefix is one contiguous slice of the sorted list, while `*pads` has to check every term.

To avoid reading the whole dictionary from disk, Lucene keeps a small **term index** on top of it. It's stored as an **FST** (finite state transducer), a compact automaton that shares prefixes and suffixes between terms. The FST points to the disk block where a term lives, and Lucene then scans only that block.

**2. Postings lists.** For each term, the **postings list** is the sorted list of document IDs containing that term. Depending on what the field is configured to keep, each entry can also hold:

| Stored per entry | Used for | `index_options` level |
|---|---|---|
| Doc ID | "does it match?" | `docs` |
| Term frequency (how often in that doc) | scoring (lesson 13) | `freqs` |
| Positions (word offsets 0, 1, 2…) | phrase queries (lesson 17) | `positions` (default for `text`) |
| Character offsets (start/end) | fast highlighting (lesson 26) | `offsets` |

`keyword` fields default to `docs`, since a whole-value exact match needs neither frequencies nor positions.

The dictionary also stores per-term statistics, most importantly **document frequency** (`doc_freq`, how many docs contain the term). That number drives IDF in scoring later.

**Doc IDs are internal.** The IDs in postings lists are not your `_id` strings. They are small integers that are local to a segment (0, 1, 2…). Because postings are sorted, Lucene stores the *gaps* between IDs instead of the IDs themselves (`3, 7, 8, 20` becomes `3, 4, 1, 12`) and bit-packs them in blocks of 128. Small gaps need few bits, so even huge postings lists stay compact and fast to decode.

**Answering a query.** A query is turned into terms, each term's postings list is fetched, and the lists are combined:
- **OR** (any term): merge the lists (union).
- **AND** (all terms): intersect them. Lucene starts from the shortest list and *skips ahead* in the longer ones, using skip data stored alongside the postings, so it never decodes the parts that can't match.

## Concrete example

Three documents in a `name` field (standard analyzer):

| Doc ID | `name` |
|---|---|
| 0 | Brake Pads for VW Golf 7 |
| 1 | Brake Disc VW Passat |
| 2 | Oil Filter for Golf |

The inverted index (dictionary sorted; postings shown as `doc(positions)`):

| Term | doc_freq | Postings |
|---|---|---|
| `7` | 1 | 0(5) |
| `brake` | 2 | 0(0), 1(0) |
| `disc` | 1 | 1(1) |
| `filter` | 1 | 2(1) |
| `for` | 2 | 0(2), 2(2) |
| `golf` | 2 | 0(4), 2(3) |
| `oil` | 1 | 2(0) |
| `pads` | 1 | 0(1) |
| `passat` | 1 | 1(3) |
| `vw` | 2 | 0(3), 1(2) |

Queries:
- `vw golf`, OR: `{0,1} ∪ {0,2}` = **0, 1, 2**. Doc 0 contains both terms, so it scores highest.
- `vw golf`, AND (`"operator": "and"`): `{0,1} ∩ {0,2}` = **0**.
- Phrase `"vw golf"`: doc 0 has `vw` at position 3 and `golf` at 4, which are adjacent, so it matches. Doc 2 has no `vw` at all.

See the real index with the term vectors API, which shows a document's terms with positions, offsets and statistics:

```http
PUT /parts-lesson2
{ "mappings": { "properties": { "name": { "type": "text" } } } }

POST /parts-lesson2/_bulk?refresh=true
{ "index": { "_id": "a" } }
{ "name": "Brake Pads for VW Golf 7" }
{ "index": { "_id": "b" } }
{ "name": "Brake Disc VW Passat" }
{ "index": { "_id": "c" } }
{ "name": "Oil Filter for Golf" }

# Terms of doc "a", with positions, offsets and doc_freq per term
GET /parts-lesson2/_termvectors/a
{
  "fields": ["name"],
  "positions": true,
  "offsets": true,
  "term_statistics": true,
  "field_statistics": true
}

# OR vs. AND over the same postings lists
GET /parts-lesson2/_search
{ "query": { "match": { "name": "vw golf" } } }

GET /parts-lesson2/_search
{ "query": { "match": { "name": { "query": "vw golf", "operator": "and" } } } }

DELETE /parts-lesson2
```

In the `_termvectors` response, each term under `terms` has `doc_freq` (e.g. `brake: 2`), `ttf` (total occurrences across the index) and a `tokens` array with `position`, `start_offset` and `end_offset`. That is the postings data for that one document. With a single shard, as here, the statistics match the table above.

## Common pitfalls
- **Leading wildcards and `*term*` queries.** They can't use the sorted dictionary, so they check every term in the field. On large fields that is slow. Use n-grams (lesson 10) or the `wildcard` field type instead.
- **Expecting the inverted index to return a field's value.** It answers "term → docs", not "doc → value". Sorting and aggregations use a separate column-oriented structure, `doc_values` (lesson 23).
- **Assuming deletes shrink the index right away.** A deleted document is only marked as deleted. Its entries stay in the postings until segments are merged (lesson 20), and they still count in `doc_freq` until then.
- **Turning off positions to save space, then needing phrases.** With `index_options: freqs` or `docs`, `match_phrase` on that field fails with an error. Changing `index_options` later needs a reindex.

## Quick quiz
1. Using the example index, which documents does `brake AND golf` return, and which postings lists get intersected?
2. Why is a `prefix` query on `"pa"` much cheaper than a `wildcard` query on `"*at"`?
3. What extra data must a postings list hold for `match_phrase` to work, and which `index_options` value provides it?

## Tomorrow
Lesson 3: the **analysis pipeline**, the char filters → tokenizer → token filters chain that decides which terms end up in this dictionary at all.

---

<details>
<summary><strong>Answers</strong></summary>

1. Only doc **0**. `brake` → {0, 1} intersected with `golf` → {0, 2} gives {0}.
2. The term dictionary is sorted, so all terms starting with `pa` (`pads`, `passat`) form one contiguous range that Lucene can jump to. `*at` can end any term, so every term in the dictionary has to be checked.
3. **Positions**, so the engine can verify the terms appear next to each other in the right order. That is `index_options: positions`, the default for `text` fields (`offsets` includes positions too).

</details>
