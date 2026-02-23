# BM25 Agent Recipe (Tiny-Model + Low-Hardware)

This site provides a **static BM25 inverted index** over `/exports/chunks.ndjson` so small agents can do strong retrieval without embeddings.

## Files
- `/exports/index/index_meta.json` — parameters + sharding rule
- `/exports/index/doc_index.json` — `doc_idx → chunk_id, url, sha256, len`
- `/exports/index/doclens.json` — `doc_lens[]` and `avgdl`
- `/exports/index/postings_shard_{00..15}.json` — term → postings

## Query → Retrieval → Summary

### 1) Tokenize query
Use the same tokenizer as the index:
- lowercase
- tokens match regex: `[a-z0-9]+`

### 2) Determine which shard(s) to fetch
Shard ID = `int(sha1(term)[:8], 16) % 16`

Fetch only those shard files:
- `/exports/index/postings_shard_XX.json`

### 3) BM25 score candidates
For each query term:
- look up `df`, `idf`, and `postings` = `[[doc_idx, tf], ...]`
- accumulate BM25 per document:

`score += idf * ((tf*(k1+1)) / (tf + k1*(1 - b + b*dl/avgdl)))`

Where:
- `k1 = 1.2`
- `b = 0.75`
- `dl = doc_lens[doc_idx]` (from `doclens.json`)
- `avgdl` (from `doclens.json`)

### 4) Pick top-K chunks
Pick K based on your context budget (typical K=4..12).

Map `doc_idx → chunk_id` using `doc_index.json`.

### 5) Fetch evidence + summarize (constrained)
Fetch chunk texts via `/exports/chunks.ndjson` (or fetch markdown section pages by `url`).

**Constraint (recommended):**
- every paragraph must cite the `chunk_id`s it used
- if evidence is missing: output `INSUFFICIENT_EVIDENCE`

## Minimal answer format (good for tiny models)
- 3–8 bullets
- each bullet ends with: `CITES: [chunk_id, ...]`
