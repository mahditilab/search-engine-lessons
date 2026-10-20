# Search Engine Internals

A daily, bite-sized course on how search engines work under the hood (Lucene, Elasticsearch, OpenSearch), written for backend engineers. One lesson a day, ~5–8 minutes each.

**Get notified:** click **Watch → Custom → Releases** on this repo. Every new lesson is published as a release.

## Curriculum

- [x] 1. [What a search engine is: documents, fields, index vs. database, full-text vs. exact match](lessons/01-what-is-a-search-engine.md) — 2026-10-20
- [ ] 2. The inverted index: terms, postings lists, term dictionary
- [ ] 3. The analysis pipeline overview: char filters → tokenizer → token filters
- [ ] 4. Character filters (html_strip, mapping, pattern_replace)
- [ ] 5. Tokenizers: standard, whitespace, keyword, letter, pattern, UAX URL/email
- [ ] 6. Token filters: lowercase, ASCII folding, stop words
- [ ] 7. Stemming: Porter/Snowball, light vs. aggressive stemmers, over/under-stemming
- [ ] 8. Lemmatization vs. stemming; language-specific analyzers (German: decompounding, umlauts)
- [ ] 9. Synonyms: index-time vs. search-time, synonym_graph
- [ ] 10. N-grams and edge n-grams (autocomplete, partial matching)
- [ ] 11. Index analyzer vs. search analyzer; the `_analyze` API
- [ ] 12. Mappings: text vs. keyword, multi-fields, dynamic mapping pitfalls
- [ ] 13. Relevance basics: TF, IDF, field length normalization
- [ ] 14. BM25 in detail (k1, b)
- [ ] 15. Query DSL: match, term, multi_match, bool (must/should/filter/must_not)
- [ ] 16. Query context vs. filter context; caching
- [ ] 17. Phrase queries, slop, positions and offsets
- [ ] 18. Fuzzy matching: edit distance, Levenshtein automata, prefix_length
- [ ] 19. Boosting, function_score, field weighting
- [ ] 20. Segments, immutability, refresh, flush, merge
- [ ] 21. Near-real-time search, translog, durability
- [ ] 22. Shards and replicas; routing; how a distributed search executes (query then fetch)
- [ ] 23. doc_values, fielddata, stored fields, _source
- [ ] 24. Aggregations: terms, histogram, cardinality (HyperLogLog)
- [ ] 25. Sorting and pagination: from/size, search_after, point-in-time
- [ ] 26. Highlighting and suggesters (term, phrase, completion)
- [ ] 27. Vector search: embeddings, kNN, HNSW
- [ ] 28. Hybrid search: combining BM25 and vectors, reciprocal rank fusion
- [ ] 29. Relevance evaluation: precision, recall, nDCG, judgment lists
- [ ] 30. Search for e-commerce: car-parts style queries (part numbers, brands, typos, German compounds) — put it all together

## Review cycle
After lesson 30, each day revisits an earlier topic in more depth.
