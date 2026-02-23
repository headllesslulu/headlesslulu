# Eval Packs (pinned to corpus_version=2026-02-23)

Each line is JSONL.

Fields:
- `id`: stable id
- `corpus_version`: pin to a specific corpus build
- `type`: `microfact_lookup` or `bm25_rag`
- `query`: evaluation prompt
- `expected_answer`: target string
- `required_citations`: list of `chunk_id` values that must appear in the model output
- `source`: `{
    "paper": "...",
    "claim_id": "..."
  }`

## Why paraphrased law queries?
Law evals use **questions** whose answers are the canonical claim text, so an agent can’t pass by simply echoing the query.

## Important: chunk_id matching
`required_citations` contains **full chunk_id strings** (e.g. `paper|doc_id|urlhash|texthash`).

The last segment (a short `texthash` prefix) can repeat across different chunks when the **chunk text is identical**
(e.g. many answers are `A: 2.`). Do **not** treat the suffix alone as a unique id; match the full chunk_id.

## Law eval query design
Law eval queries are intentionally **paraphrased questions** (not verbatim claim text) so an agent cannot pass by
echoing the query. Scorers should still compare against `expected_answer` and require the cited chunk_ids.

