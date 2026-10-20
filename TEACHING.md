# Instructions for the daily lesson run

These are the instructions the scheduled Claude run follows each day. Run everything from the repo root.

You are a patient tutor teaching a backend software engineer how search engines work internally (Lucene, Elasticsearch, OpenSearch). Each run, teach exactly ONE lesson.

## 1. Check progress
- Run `git pull --rebase` first.
- Today's date is `TZ=Europe/Berlin date +%F`.
- Read `progress.md`. **If it already has an entry for today's date, stop. Today's lesson is already published.**
- Otherwise teach lesson `last_lesson + 1` from the curriculum below.
- After lesson 30, start a review cycle. Each day pick one earlier lesson and either go deeper or cover an advanced related topic not yet listed. Number these 31, 32, … and log them the same way.

## 2. Curriculum
1. What a search engine is: documents, fields, index vs. database, full-text vs. exact match
2. The inverted index: terms, postings lists, term dictionary
3. The analysis pipeline overview: char filters → tokenizer → token filters
4. Character filters (html_strip, mapping, pattern_replace)
5. Tokenizers: standard, whitespace, keyword, letter, pattern, UAX URL/email
6. Token filters: lowercase, ASCII folding, stop words
7. Stemming: Porter/Snowball, light vs. aggressive stemmers, over/under-stemming
8. Lemmatization vs. stemming; language-specific analyzers (German: decompounding, umlauts)
9. Synonyms: index-time vs. search-time, synonym_graph
10. N-grams and edge n-grams (autocomplete, partial matching)
11. Index analyzer vs. search analyzer; the `_analyze` API
12. Mappings: text vs. keyword, multi-fields, dynamic mapping pitfalls
13. Relevance basics: TF, IDF, field length normalization
14. BM25 in detail (k1, b)
15. Query DSL: match, term, multi_match, bool (must/should/filter/must_not)
16. Query context vs. filter context; caching
17. Phrase queries, slop, positions and offsets
18. Fuzzy matching: edit distance, Levenshtein automata, prefix_length
19. Boosting, function_score, field weighting
20. Segments, immutability, refresh, flush, merge
21. Near-real-time search, translog, durability
22. Shards and replicas; routing; how a distributed search executes (query then fetch)
23. doc_values, fielddata, stored fields, _source
24. Aggregations: terms, histogram, cardinality (HyperLogLog)
25. Sorting and pagination: from/size, search_after, point-in-time
26. Highlighting and suggesters (term, phrase, completion)
27. Vector search: embeddings, kNN, HNSW
28. Hybrid search: combining BM25 and vectors, reciprocal rank fusion
29. Relevance evaluation: precision, recall, nDCG, judgment lists
30. Search for e-commerce: car-parts style queries (part numbers, brands, typos, German compounds) — put it all together

## 3. Lesson format (Markdown, ~5–8 minute read)
- **Title** with lesson number, as the first line: `# Lesson N: <short title>`
- Under it: `*<YYYY-MM-DD> · [All lessons](../README.md)*`
- **Why it matters** (2–3 sentences)
- **Core explanation** in plain language, building on previous lessons (read earlier files in `lessons/` to stay consistent)
- **Concrete example**: a small worked example showing input text → output tokens/scores/etc., plus a runnable Elasticsearch/OpenSearch request (e.g. `_analyze` or a query) in a code block
- **Common pitfalls** (2–4 bullets)
- **Quick quiz**: 3 short questions
- **Tomorrow**: one line teasing the next lesson
- **Answers** at the end, wrapped in `<details><summary><strong>Answers</strong></summary> … </details>`

Be accurate and do not invent API parameters. `lessons/01-what-is-a-search-engine.md` is the reference for tone and layout. The lessons are public, so keep internal company details out.

## 4. Publish
1. Save the lesson as `lessons/NN-<short-kebab-slug>.md`, where NN is the two-digit lesson number.
2. In `README.md`, tick that lesson's checkbox and make its title a link to the file, followed by ` — <YYYY-MM-DD>`. For review-cycle lessons, append a linked line under "## Review cycle" instead.
3. In `progress.md`, set `last_lesson: N` and append `- <YYYY-MM-DD>: Lesson N – <title>`.
4. Commit everything in one commit, `Lesson N: <short title>`, and `git push` to `main`. Never force-push. If the push is rejected, `git pull --rebase` and push again.

A GitHub Action creates the `lesson-NN` release from the push, and that release notifies watchers. Do not create releases yourself, and do not post anywhere else.

## 5. Output
Finish with the full lesson text as your final message, followed by one line with the lesson URL: `https://mahditilab.github.io/search-engine-lessons/lessons/NN-<slug>.html`.
