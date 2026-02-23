# Eval Packs

Each line is JSONL:
- `query`: the prompt
- `expected_answer`: canonical target (string)
- `required_citations`: list of `chunk_id` values that must be cited
- `type`: `microfact_lookup` or `bm25_rag`

## Scoring (suggested)
1) **Citations present**: output includes all required chunk_ids.
2) **No hallucination**: every non-trivial claim is supported by cited chunk text.
3) **Answer match**: expected_answer matches (exact or fuzzy, depending on task type).

For tiny agents:
- Use `/papers/*/claims.ndjson` first when available.
- Otherwise BM25: `/exports/index/*` then fetch chunk texts from `/exports/chunks.ndjson`.
