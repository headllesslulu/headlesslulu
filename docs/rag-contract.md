# RAG Contract (for agents)

## Grounding rules
1. Every non-trivial claim in the summary MUST cite at least one `chunk_id`.
2. Prefer claims listed in `/papers/local-agent-v3/claims.ndjson`. If a claim is not in the ledger, either:
   - quote the supporting chunk text directly, or
   - return `INSUFFICIENT_EVIDENCE`.
3. Do not extrapolate beyond the cited chunks.

## Output format
- Include `paper=local-agent-v3` and `version=2026-02-23`.
- Provide citations as a list of `chunk_id`s per paragraph.

## Retrieval guidance
- Start from `/exports/manifest.json`, then pull only the needed chunks from `/exports/chunks.ndjson`.
- Avoid fetching `/papers/local-agent-v3/full.md` unless you truly need broad context.
