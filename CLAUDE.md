# ConsistRAG

Graduation project: a consistency-aware RAG system that detects, classifies and resolves conflicting information across
sources before answering, with citations and calibrated confidence. Python backend, Next.js frontend.

The brief is `docs/project_spec.md` (cited as §N). The design is `docs/design.md`. Read the design before starting any
new module. The decisions below are fixed; do not change them without asking.

## Fixed decisions

- **Scope:** full target system (spec §32). No stretch goals (§38).
- **Language:** English only.
- **Domain:** Erasmus+ programme rules (real corpus R) + synthetic "Northbridge University" corpus S with injected
  conflicts. *Erasmus+ still to be confirmed by the user; backup is versioned software docs.*
- **LLM:** local only, via Ollama, 7–8B instruct at 4-bit (generator Qwen2.5-7B-Instruct). No paid APIs. The judge is a
  different model family from the generator. All LLM calls go through the OpenAI-compatible client in `consistrag/llm/`.
- **Hardware:** everything must run on an RTX 4060 (8 GB) + Ryzen 5 5600. One LLM loaded at a time.
- **No RAG frameworks:** no LangChain, LlamaIndex or Haystack. Own pipeline code on sentence-transformers,
  transformers, bm25s, pgvector, pydantic.
- **Storage:** PostgreSQL + pgvector for documents, chunks, claims, graph data and evaluation runs.
- **Retrieval:** bge-base-en-v1.5 dense + BM25 (bm25s) → RRF (k=60) → bge-reranker-base.
- **Claims are extracted at ingestion time**, stored, never at query time.
- **Contradiction detection is a cascade:** rules → numeric module → DeBERTa-v3 NLI → LLM only for uncertain pairs.
- **Source-score weights are learned on the dev split**, never hand-picked. Every threshold (k, θ, τ, tolerances) is
  tuned on dev only; the test split is used only for final runs.
- **Confidence:** consistency score and calibrated answer confidence are separate numbers. LLM token probabilities are
  not used as confidence.
- **Backend/frontend/deploy:** FastAPI, Next.js, Docker Compose.
- **Ground truth:** every test item carries an evidence pointer and is verified by the team (`verified_by`).

## Structure

See `docs/design.md` §8.2. `consistrag/pipeline.py: answer(question, config)` is the only entry point the API, UI and
evaluation use. Each baseline, variant and ablation is one file in `configs/`.

## Conventions

- Python 3.11+, type hints, docstrings on public functions, pydantic models for every stage's input and output.
- Every module gets pytest unit tests before the next layer is built on it. Unit tests use fakes for the LLM and NLI
  so they run on CPU-only CI.
- LLM calls: temperature 0, fixed seed, JSON-schema output where structured; responses cached by (model digest,
  prompt hash).
- Every pipeline run logs its intermediate results (trace) so failures can be traced to a stage.
- Raw third-party documents are never committed; only manifests and fetch scripts.

## Build order

0. Skeleton: layout, Docker Compose, CI.
1. Literature notes, conflict taxonomy, benchmark schema, corpus manifest, synthetic generator.
2. Baseline RAG (B1, B2).
3. Claim extraction + its evaluation.
4. Contradiction detection; compare NLI / LLM / hybrid.
5. Temporal analysis, consistency graph, source scoring.
6. Resolution, abstention, generation with citation check, confidence.
7. API + UI.
8. Full evaluation, ablations, robustness, error analysis, Docker deployment.
9. Report.
