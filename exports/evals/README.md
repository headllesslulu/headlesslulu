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
