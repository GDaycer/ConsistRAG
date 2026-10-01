# ConsistRAG

Graduation project: a consistency-aware RAG system that detects, classifies and resolves conflicting information across
sources before answering, with citations and calibrated confidence. Python backend, Next.js frontend.

The brief is `docs/project_spec.md` (cited as §N). The design is `docs/design.md`. Read the design before starting any
new module. The decisions below are fixed; do not change them without asking.

## Fixed decisions

- **Scope:** full target system (spec §32). No stretch goals (§38).
- **Language:** English only.
- **Domain:** Erasmus+ programme rules (real corpus R) + synthetic "Northbridge University" corpus S with injected
  conflicts.
- **LLM:** local only, via Ollama, 7–8B instruct at 4-bit (generator Qwen2.5-7B-Instruct). No paid APIs. The judge is a
  different model family from the generator. All LLM calls go through the OpenAI-compatible client in `consistrag/llm/`.
- **Hardware:** development and experiments on an RTX 4060 (8 GB) + Ryzen 5 5600, one LLM loaded at a time. The
  deployed system must also run **without an NVIDIA GPU** via the CPU profile (3B model, NLI base,
  no LLM step in the cascade). Profiles are config values; the code path is the same.
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
- **Backend/frontend:** FastAPI, Next.js.
- **Deployment:** Docker Compose (`docker compose up` = CPU profile; add `docker-compose.gpu.yml` for GPU). The project
  must be redeployable on a new machine from the README alone, either by re-ingesting or by restoring the released
  database dump. Versions are pinned; outputs may differ slightly between machines.
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

See `docs/design.md` §9 (phases, exit checks) and §9.1 (deliverable traceability).

0. Skeleton: layout, Docker Compose, start scripts, CI.
1. Literature review and requirements (D1).
2. Baseline RAG + evaluation runner (D3).
3. Benchmark, team-verified (D2).
4. Claim extraction.
5. Contradiction detection; compare NLI / LLM / hybrid.
6. Temporal analysis, consistency graph, source scoring, conflict types (D4).
7. Resolution, abstention, generation with citation check, confidence (D5).
8. API + UI (D6).
9. Full evaluation, ablations, robustness, error analysis (D7).
10. Deployment: database dump release, README, redeployment test on a fresh machine (D8).
11. Report (D9).
