# Lesson 4: Character filters

*2026-10-23 · [All lessons](../README.md)*

## Why it matters
Lesson 3 showed the analysis chain; the character filters are its first stage. They fix the raw string *before* the tokenizer splits it, so they are the right tool for problems the tokenizer can't see: HTML markup in descriptions, `&` that should mean "and", part numbers written as `0 986-494.104` in one system and `0986494104` in another. One small char filter can remove a whole class of "why doesn't this match?" bugs.

## Core explanation

A **character filter** takes the whole text as a string and returns a (possibly different) string. It knows nothing about words or tokens. Several can be chained, and they run in the order you list them, each one seeing the output of the previous one. Elasticsearch/OpenSearch ship three:

**`html_strip`**: removes HTML tags and decodes HTML entities (`&amp;` → `&`, `&uuml;` → `ü`). An optional `escaped_tags` setting lists tags that should be left in place. Use it for product descriptions that come from a CMS or a supplier feed.

**`mapping`**: replaces fixed strings with other strings. You give it a list of `"from => to"` rules (setting `mappings`, or a file via `mappings_path`). The longest matching `from` wins. Typical uses: `& => and`, `ä => ae`, `ß => ss`, or turning `½` into `0.5`.

**`pattern_replace`**: replaces whatever a Java regular expression matches (settings `pattern`, `replacement`, optional `flags`). The replacement can refer to capture groups as `$1`, `$2`. This is the most flexible one and the easiest to get wrong.

**Offsets still point at the original text.** Because the string is rewritten before tokenizing, token offsets would drift away from the stored `_source`. Lucene's char filters keep an offset-correction map, so highlighting (lesson 26) still marks the right characters in the original text. Positions are unaffected too.

**Char filter or token filter?** If the change needs to be seen *before* word boundaries are decided (`0 986-494.104` should become one token, not four), it must be a char filter. If it only changes individual tokens (lowercasing, stemming), use a token filter (lesson 6 onward). Char filters are also a bit more expensive, since they scan the whole string, so don't use them for what a token filter does just as well.

## Concrete example

Input: `<p>Bremsscheibe VW &amp; Audi 0 986-494.104</p>`

Chain: `html_strip` → `mapping` (`& => and`) → `pattern_replace` (remove a space, dot or hyphen *between two digits*) → `standard` tokenizer → `lowercase`.

| Stage | Result |
|---|---|
| after `html_strip` | `Bremsscheibe VW & Audi 0 986-494.104` |
| after `mapping` | `Bremsscheibe VW and Audi 0 986-494.104` |
| after `pattern_replace` | `Bremsscheibe VW and Audi 0986494104` |
| tokens after `standard` + `lowercase` | `bremsscheibe`, `vw`, `and`, `audi`, `0986494104` |

Without the `pattern_replace` the standard tokenizer would produce `0`, `986`, `494`, `104` (it splits on the space and, between digits, on `-`), and a user typing `0986494104` would find nothing. The regex `(\d)[\s.\-](?=\d)` with replacement `$1` keeps the digit before the separator and drops the separator, but only when another digit follows, so `VW-Golf` and normal sentences stay untouched.

Run it:

```http
POST /_analyze
{
  "char_filter": [
    "html_strip",
    { "type": "mapping", "mappings": ["& => and"] },
    { "type": "pattern_replace", "pattern": "(\\d)[\\s.\\-](?=\\d)", "replacement": "$1" }
  ],
  "tokenizer": "standard",
  "filter": ["lowercase"],
  "text": "<p>Bremsscheibe VW &amp; Audi 0 986-494.104</p>",
  "explain": true
}
```

And as a reusable analyzer:

```http
PUT /parts-lesson4
{
  "settings": {
    "analysis": {
      "char_filter": {
        "ampersand_to_and": { "type": "mapping", "mappings": ["& => and"] },
        "join_digit_groups": {
          "type": "pattern_replace",
          "pattern": "(\\d)[\\s.\\-](?=\\d)",
          "replacement": "$1"
        }
      },
      "analyzer": {
        "parts_text": {
          "type": "custom",
          "char_filter": ["html_strip", "ampersand_to_and", "join_digit_groups"],
          "tokenizer": "standard",
          "filter": ["lowercase"]
        }
      }
    }
  },
  "mappings": {
    "properties": { "name": { "type": "text", "analyzer": "parts_text" } }
  }
}

POST /parts-lesson4/_analyze
{ "analyzer": "parts_text", "text": "<p>Bremsscheibe VW &amp; Audi 0 986-494.104</p>" }

DELETE /parts-lesson4
```

## Common pitfalls
- **Over-eager regexes.** A pattern that joins digit groups will also turn `Golf 7 2019` into `Golf 72019`. Make the pattern as narrow as the data allows and test it on realistic inputs with `_analyze`.
- **Regex syntax.** `pattern_replace` uses Java regex. In JSON every backslash must be doubled (`\\d`), and `$1` is the group reference, not `\1`.
- **Rules that depend on order.** `mapping` rules run on what `html_strip` produced, so `&amp;` has to be decoded first (list `html_strip` first) or your `& => and` rule never sees an `&`.
- **Doing token work in a char filter.** Lowercasing or accent folding is cheaper and clearer as a token filter. Reserve char filters for changes that must happen before tokenizing.

## Quick quiz
1. Why can't a token filter turn `0 986-494.104` into the single token `0986494104`?
2. You list the char filters as `["mapping", "html_strip"]` with the rule `& => and`. What happens to `Tom &amp; Jerry`?
3. Does a char filter that rewrites the text break highlighting on the original `_source`? Why or why not?

## Tomorrow
Lesson 5: **tokenizers**: `standard`, `whitespace`, `keyword`, `letter`, `pattern` and the URL/email tokenizer, the component that finally splits the string into tokens.

---

<details>
<summary><strong>Answers</strong></summary>

1. By the time token filters run, the tokenizer has already split the text into `0`, `986`, `494`, `104`. A token filter sees single tokens and cannot merge neighbours back together. The separators have to be removed before tokenizing, which is the job of a char filter.
2. `mapping` runs first and sees the raw text `Tom &amp; Jerry`. Its rule replaces the `&` inside `&amp;`, giving `Tom andamp; Jerry`. Then `html_strip` finds no entity left to decode. Put `html_strip` first.
3. No. Lucene char filters record offset corrections, so token offsets are mapped back to positions in the original string and the highlighter marks the right characters.

</details>
