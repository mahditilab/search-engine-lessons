# Lesson 6: Token filters: lowercase, ASCII folding, stop words

*2026-11-10 · [All lessons](../README.md)*

## Why it matters
The tokenizer (lesson 5) only cuts text into pieces. Token filters decide what those pieces look like in the term dictionary: `Škoda`, `skoda` and `SKODA` are three different terms unless a filter makes them one. Lowercasing, folding accents and dropping noise words are the cheapest wins in search relevance, and getting them slightly wrong is a classic source of silent misses.

## Core explanation

In the pipeline from lesson 3, token filters are the last stage: **zero or more**, run **in the order you list them**, each one receiving the token stream from the previous one. A filter can **change** tokens (lowercase), **remove** them (stop words), or **add** them (synonyms, lesson 9). Today we cover the three you will put in almost every analyzer.

### `lowercase`
Turns every token into lower case, so `Brake`, `BRAKE` and `brake` all become the term `brake`. Without it, the case of the query must match the case of the document exactly (remember: tokenizers keep case). It has an optional `language` parameter (`greek`, `irish`, `turkish`) for languages with special casing rules. The Turkish dotted/dotless `i` is the famous case where the default rules give the wrong result. There is also an `uppercase` filter, which you will rarely want.

### `asciifolding`
Maps characters outside basic ASCII to their closest ASCII equivalent: `é` → `e`, `Š` → `S`, `ø` → `o`, `æ` → `ae`, `ß` → `ss`, and German umlauts `ü` → `u`. That way `Škoda` and `Skoda`, or `Citroën` and `Citroen`, meet in the dictionary.

- `preserve_original: true` emits **both** the folded and the original token at the same position. `café` produces `cafe` and `café`. Exact spellings can then still score a little higher, at the cost of a bigger index.
- Folding is lossy by design. It does not know languages: `ü` becomes `u`, not the German `ue`. German-specific handling (`ue` ↔ `ü`, decompounding) comes in lesson 8.

### `stop`
Removes tokens that appear in a stop word list. The idea: words like `the`, `of` and `for` appear in nearly every document, so they barely help ranking (lesson 13 shows why) but cost space in postings lists.

- `stopwords`: a predefined list such as `_english_` or `_german_`, an explicit array like `["the", "for"]`, or `_none_`. The default is `_english_`.
- `stopwords_path`: a file of stop words (one per line) on each node.
- `ignore_case` (default `false`): by default the match is case-sensitive, which is why `stop` belongs **after** `lowercase`.
- Removed tokens leave **position gaps** (lesson 3), so phrase queries still respect the original distances.

The built-in `_english_` list is short: `a, an, and, are, as, at, be, but, by, for, if, in, into, is, it, no, not, of, on, or, such, that, the, their, then, there, these, they, this, to, was, will, with`. Note `no` and `not` are on it.

Modern Elasticsearch and OpenSearch apply **no** stop filter in the default `standard` analyzer. You opt in. With BM25 scoring and fast postings, many teams leave stop words in and let IDF downweight them.

### A typical order
`lowercase` → `asciifolding` → `stop` → (stemming, lesson 7). Lowercase first, so the case-sensitive stop list matches. Folding before stop words means your stop list must contain folded spellings for languages that have accents (the `_german_` list includes `für`, which `asciifolding` would already have turned into `fur`).

## Concrete example

Input: `Bremsbeläge für Škoda Fabia, NOT for the Octavia`

| Stage | Tokens (position) |
|---|---|
| `standard` tokenizer | `Bremsbeläge`(0) `für`(1) `Škoda`(2) `Fabia`(3) `NOT`(4) `for`(5) `the`(6) `Octavia`(7) |
| `lowercase` | `bremsbeläge` `für` `škoda` `fabia` `not` `for` `the` `octavia` |
| `asciifolding` | `bremsbelage` `fur` `skoda` `fabia` `not` `for` `the` `octavia` |
| `stop` (`_english_`) | `bremsbelage`(0) `fur`(1) `skoda`(2) `fabia`(3) `octavia`(7) |

Now a query for `skoda octavia` matches this document, even though the text explicitly says it does *not* fit the Octavia. The stop filter removed `not`. Also note that the German `für` survived, because the English list doesn't know it.

Try it (no index needed). `explain: true` shows each filter's output:

```http
POST /_analyze
{
  "tokenizer": "standard",
  "filter": ["lowercase", "asciifolding", "stop"],
  "text": "Bremsbeläge für Škoda Fabia, NOT for the Octavia",
  "explain": true
}

# Keep the original next to the folded token
POST /_analyze
{
  "tokenizer": "standard",
  "filter": ["lowercase", { "type": "asciifolding", "preserve_original": true }],
  "text": "Citroën Škoda"
}
```

The second call returns `citroen` and `citroën` at position 0, `skoda` and `škoda` at position 1.

A reusable analyzer with a custom stop list that keeps `not`:

```http
PUT /parts-lesson6
{
  "settings": {
    "analysis": {
      "filter": {
        "parts_stop": {
          "type": "stop",
          "stopwords": ["a", "an", "and", "the", "for", "of", "fur"]
        }
      },
      "analyzer": {
        "parts_text": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding", "parts_stop"]
        }
      }
    }
  },
  "mappings": {
    "properties": { "name": { "type": "text", "analyzer": "parts_text" } }
  }
}

POST /parts-lesson6/_analyze
{ "analyzer": "parts_text", "text": "Bremsbeläge für Škoda Fabia, NOT for the Octavia" }

DELETE /parts-lesson6
```

Result: `bremsbelage`, `skoda`, `fabia`, `not`, `octavia`. The folded `fur` is gone because the custom list contains the folded spelling.

## Common pitfalls
- **`stop` before `lowercase`.** `The` and `NOT` slip through the case-sensitive list. Put `lowercase` first or set `ignore_case: true`.
- **Stop words that carry meaning.** `not`, `no`, or `a` in "Vitamin A" or "Type A". Removing them changes the meaning or makes some queries impossible. A phrase search for `to be or not to be` with `_english_` stop words has nothing left to match.
- **Folding on only one side.** If the index folds `Škoda` to `skoda` but the search analyzer doesn't fold, the query term `škoda` finds nothing. Use the same normalization at index and search time (lesson 11).
- **Expecting `asciifolding` to do German right.** `Müller` becomes `muller`, but a user typing `Mueller` gets `mueller`, a different term. That needs a German-aware filter (lesson 8).

## Quick quiz
1. With the filter chain `["stop", "lowercase"]` and the default `_english_` list, what happens to the token `For`?
2. What does `asciifolding` with `preserve_original: true` emit for `Peugeot Citroën`, and at which positions?
3. Why might you deliberately leave stop words in your index with BM25?

## Tomorrow
Lesson 7: **stemming**: how Porter and Snowball turn `brakes` and `braking` into `brake`, and when they go too far.

---

<details>
<summary><strong>Answers</strong></summary>

1. It survives as `for`. The stop filter runs first and matches case-sensitively, so `For` is not in the list. Then `lowercase` turns it into `for`, which is now indexed.
2. `Peugeot` at position 0 (pure ASCII, so nothing to fold and only one token), and `Citroen` plus `Citroën` both at position 1. Add `lowercase` before it and you get `peugeot`, `citroen`, `citroën`.
3. IDF already gives very common terms a tiny weight, so they hardly affect ranking. Keeping them lets phrase queries and meaningful words like `not` or `A` still work, and modern postings compression keeps the size cost small.

</details>
