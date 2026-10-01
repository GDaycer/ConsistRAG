# ConsistRAG — Design

Consistency-aware Retrieval-Augmented Generation over conflicting knowledge sources.
The original brief is in `project_spec.md`; section numbers written as §N refer to it.
This document records how we meet that brief. Decisions marked **fixed** are also listed in `CLAUDE.md`
and change only by agreement.

---

## 0. Decisions at a glance

| Topic | Decision | Status |
|---|---|---|
| Scope | Full target system (§32). No stretch goals (§38). | fixed |
| Language | English documents, questions and answers. | fixed |
| Domain | Erasmus+ programme rules (EU, 2021–2027 programme, yearly versions), plus a synthetic fictional-university corpus. | fixed |
| LLM | Local, via Ollama, 7–8B instruct model at 4-bit. No paid API anywhere. | fixed |
| Hardware target | Development and experiments: RTX 4060 8 GB, Ryzen 5 5600. Distribution: any Windows 10/11 PC, **with or without an NVIDIA GPU** (CPU profile, §8.6). | fixed |
| Framework | Own pipeline code on standard libraries. No LangChain / LlamaIndex / Haystack. | fixed |
| Storage | PostgreSQL 16 + pgvector (documents, chunks, embeddings, claims, graph, eval runs). | fixed |
| Backend / frontend | FastAPI / Next.js (React). | fixed |
| Deployment | Docker Compose on Windows (Docker Desktop + WSL2). Goal: a working program on ~20 machines from one README, not identical outputs. Prebuilt index shipped; no machine re-ingests. | fixed |
| Ground truth | Built by Claude, verified by the team (§22 "manually verified"). | fixed |

---

## 1. Domain and corpus

### 1.1 Why Erasmus+

The brief asks for "public documents from domains where multiple versions exist" (§21.2), and its examples are
university rules (attendance, tuition, programme duration). Erasmus+ fits this closely and is relevant to students in
Türkiye, which takes part in the programme:

- **Many versions:** the European Commission publishes a new *Erasmus+ Programme Guide* for every call year
  (2021 … 2026), plus corrigenda. Rules and amounts change between years. This gives **temporal** and **version** conflicts.
- **Numbers:** grant rates, travel bands, minimum and maximum mobility durations, age limits and deadlines.
  This gives **numerical** conflicts.
- **Scope:** rules differ by action (study vs. traineeship, short vs. long-term mobility, student vs. staff,
  country group, green travel). This gives **scope** conflicts that are not real contradictions.
- **Sources of different authority:** the EU Regulation and the Programme Guide (official), Commission and National
  Agency pages, university international-office pages, news, and student blogs and forums that quote old figures. This gives
  **source-reliability** conflicts.
- **Clear dates:** each guide states the call year it applies to, which gives reliable effective dates.

Backup domain if Erasmus+ is rejected: versioned software documentation (e.g. Python or Django docs, release notes, deprecations
and Stack Overflow answers). The pipeline is domain-independent. Only the corpus manifest, the authority table (§4.6)
and the benchmark change.

### 1.2 Real corpus (Corpus R)

Target: 150–300 documents. Each one is listed in `data/manifests/corpus_r.yaml` with:

```yaml
- doc_id: PG2024
  url: https://...
  title: Erasmus+ Programme Guide 2024
  publisher: European Commission        # used for independence checks
  source_type: official_guide           # maps to authority tier (§4.6)
  publication_date: 2023-11-28
  effective_from: 2024-01-01            # when the content starts to apply, if known
  effective_to: 2024-12-31              # optional
  version: "2024 v1"                    # optional, links versions of one document lineage
  lineage: programme_guide              # documents in the same lineage can supersede each other
  scope_tags: [higher_education]
  license: "EU reuse (Decision 2011/833/EU)"
  archive_url: https://web.archive.org/web/...   # snapshot we used, in case the page changes
  sha256: ...                                     # of the fetched file
```

**Licensing:** EU documents may be reused with attribution. For third-party pages (universities, news, forums), the
repo commits only the manifest and a fetch script. Raw text is stored locally under `data/raw/` (gitignored), so we do
not republish other people's content.

### 1.3 Synthetic corpus (Corpus S)

A fictional institution, *Northbridge University*, generated from templates by `data/synth/generate.py` with a fixed
seed. It mirrors the demo scenarios in §37: attendance requirement, tuition fee, programme duration and eligibility
threshold. Every document is a template filled from a fact table, so the ground truth is known for certain. Conflicts
are injected on purpose:

| Injection | Example |
|---|---|
| Version | Regulation v1 (2024) says 70 %; v2 (2026) says 80 %. |
| Temporal | "From January 2026 the warranty / fee changed from X to Y." |
| Numerical | Two pages give 240,000 vs. 245,000; also rounding cases (< 1 %). |
| Scope | 70 % for lecture courses vs. 80 % for laboratory courses. |
| Reliability | Official regulation 80 % vs. forum post 70 %. |
| Unresolvable | Two equally authoritative current documents disagree. |
| No conflict | Paraphrased, agreeing sources (controls). |

We also inject a few controlled perturbations into Corpus R (e.g. a fake blog quoting an old grant rate), marked
`synthetic: true` in the manifest.

---

## 2. Architecture

The work is split into an **offline ingestion path** that does the expensive work once, and an **online query path**
that must answer in a few seconds on a 4060. Claim extraction runs at ingestion time and its results are stored. At 30–50
tokens/s for a 7B model, extracting claims at query time would be too slow.

```text
INGESTION (offline, once per document)
  fetch → text extraction → metadata → section-aware chunking
        → embeddings (pgvector) + BM25 index
        → claim extraction (LLM, JSON schema) → claim normalisation
        → stored: documents, chunks, claims

QUERY (online)
  question
    → query analysis           (temporal intent, as_of date, scope hints, answer type)
    → hybrid retrieval         (dense + BM25 → RRF) → cross-encoder rerank → top-k chunks
    → evidence claims          (stored claims of those chunks, filtered for relevance)
    → candidate pairing        (blocking by subject/attribute)
    → relation classification  (rules → numeric → NLI → LLM, cascade)
    → consistency graph        (claims, documents, sources; supports/contradicts/supersedes/…)
    → temporal analysis        (validity intervals, CURRENT / OUTDATED / FUTURE / UNKNOWN)
    → source scoring           (authority, recency, relevance, specificity, independent support)
    → resolution               (resolved / contextual / unresolved / insufficient)
    → answer generation        (structured brief → LLM → citation check)
    → confidence estimation    (consistency score + calibrated confidence)
    → ANSWER + CITATIONS + CONFIDENCE + CONFLICT REPORT
```

Every stage is a plain Python function with typed input and output (pydantic models), selected and configured through a
**pipeline config** (`configs/*.yaml`). Each baseline, variant and ablation in §5 is one config file. Every
intermediate result is logged to the `run_traces` table so error analysis (§7) can trace each failure to the stage that caused it.

---

## 3. Baseline RAG (Deliverable 3)

This shared retrieval code is used by both the baseline and ConsistRAG, so the comparison is fair.

| Step | Choice |
|---|---|
| Text extraction | PyMuPDF for PDF, trafilatura for HTML. Keep heading structure. |
| Chunking | Section-aware: split on headings, then on ~350 tokens with 50-token overlap. Each chunk keeps its section path (e.g. "Part B › Mobility of students › Grant rates"). Tables are kept whole and converted to text. |
| Embeddings | `BAAI/bge-base-en-v1.5` (768-d), stored in pgvector with an HNSW index. |
| Sparse | BM25 with `bm25s`, built in memory from the chunk table at startup (the corpus is small). Postgres `ts_rank` is not BM25, so we do not use it. |
| Fusion | Reciprocal Rank Fusion, k = 60, top-50 from each retriever. |
| Reranker | Cross-encoder `BAAI/bge-reranker-base`, top-50 → top-k (k = 8 by default, tuned on the dev split). |
| Generation | Numbered passages [S1]…[Sk] with title and date, then "answer using only these and cite [S#]". |

---

## 4. Consistency layer

### 4.1 Claim schema and extraction (Phase 3)

The spec's representation (§7), extended with the fields the later stages need:

```json
{
  "claim_id": "C102",
  "chunk_id": "PG2024#c311",
  "doc_id": "PG2024",
  "text": "Students on long-term study mobility receive a monthly grant of EUR X in country group 1.",
  "subject": "erasmus+ student mobility grant",
  "attribute": "monthly_amount",
  "value": {"raw": "EUR X", "type": "money", "amount": 0, "unit": "EUR", "per": "month"},
  "qualifiers": {"scope": ["long_term", "study", "country_group_1"], "condition": null},
  "time": {"valid_from": "2024-01-01", "valid_to": null, "source": "doc_effective_date"},
  "polarity": "positive",
  "extraction_confidence": 0.94
}
```

- **Extractor:** the local LLM with Ollama structured output (JSON schema), temperature 0 and few-shot examples from the
  domain. One call per chunk returns a list of claims.
- **Normaliser** (deterministic code, not the LLM):
  - parses values: money with currency, percentages, durations, dates, counts and ranges
  - canonicalises `subject` and `attribute` using a small alias table plus embedding similarity, so that
    "monthly grant", "grant per month" and "monthly support" map to the same attribute
- **Fallback:** if extraction fails or the JSON is invalid, each sentence becomes a claim with only `text` set. NLI can
  still compare it.
- **Evaluation:** 50 hand-labelled chunks; claim-level precision and recall (a match is judged by NLI entailment in both
  directions plus a human spot check); value-parsing accuracy.

### 4.2 Candidate pairing

Comparing every claim with every other one is O(n²) and too slow for NLI or LLM calls. We pair only claims that share a canonical
`(subject, attribute)` or have cosine similarity ≥ 0.75. A typical query then has 10–60 pairs instead of thousands.

### 4.3 Relation classification (hybrid cascade, §10 Approach C)

Pairwise labels (the spec's §8 classes):

`ENTAILMENT · CONTRADICTION · NUMERICAL_CONFLICT · TEMPORAL_CONFLICT · PARTIAL_CONFLICT · CONTEXT_DEPENDENT · UNRELATED · UNKNOWN`

The cascade runs the cheapest and most precise steps first:

1. **Rules.**
   - Different, compatible scope qualifiers → `CONTEXT_DEPENDENT`.
   - Validity intervals that do not overlap and the same lineage → `TEMPORAL_CONFLICT`.
   - Explicit "replaced", "from … changed to …" → `TEMPORAL_CONFLICT` plus a `supersedes` edge.
2. **Numeric module (§11).** Both values parsed and of the same kind:
   - convert units, then compute the absolute and relative difference
   - equal → `ENTAILMENT`
   - relative difference < 1 % → `ENTAILMENT` flagged `POSSIBLE_ROUNDING`
   - overlapping ranges → `PARTIAL_CONFLICT`
   - otherwise → `NUMERICAL_CONFLICT`
   - the tolerance is a config value, checked on the dev split
3. **NLI.** `MoritzLaurer/DeBERTa-v3-large-mnli-fever-anli-ling-wanli` (fall back to the base size if VRAM is short), run in
   both directions. Contradiction in either direction ≥ θc → `CONTRADICTION`; entailment in both directions →
   `ENTAILMENT`; one direction only → `PARTIAL_CONFLICT`; otherwise `UNRELATED`.
4. **LLM classifier (§10 Approach B).** Called only when steps 1–3 are uncertain (NLI max-probability < θu, or rules and NLI
   disagree). It returns `{relationship, confidence, explanation}` as JSON. In the "LLM-only" variant (§5) it classifies every
   pair, so that we can compare the approaches.

θc and θu are tuned on the dev split of the pair set (§6.2).

### 4.4 Consistency graph (§15)

A `networkx` graph is built per query; there is also an optional persistent version for the conflict explorer in the UI.

- Nodes: claims, documents, sources (publishers).
- Edges: `supports`, `contradicts` (with the relation label), `supersedes`, `same_as`, `derived_from` (a document that quotes or copies
  another; detected by near-duplicate text and by explicit citations), `temporally_precedes`.

Claims joined by `same_as` or `supports` form **value clusters**. These are the candidate answers that resolution chooses between.

### 4.5 Temporal analysis (§12)

- Each claim gets a validity interval `[valid_from, valid_to)`. We take the first available of:
  explicit text ("from 1 January 2026", "for the 2024 call") → document `effective_from/to` → version order in the lineage
  → `publication_date` as a lower bound → unknown.
- The query analyser sets `as_of`: an explicit year or date in the question, otherwise today (configurable for
  reproducible runs).
- Each claim is then labelled `CURRENT`, `OUTDATED`, `FUTURE` or `UNKNOWN` relative to `as_of`.
- `supersedes` edges: a newer document in the same lineage with the same `(subject, attribute, scope)` supersedes the
  older claim.

### 4.6 Source reliability (§13, §14)

The spec warns against unjustified weights. Our approach:

**Authority:** an ordinal tier with a documented rationale for each tier:

| Tier | Source types | Prior |
|---|---|---|
| A | EU Regulation, official Programme Guide | 1.00 |
| B | Commission or National Agency pages, official university regulations | 0.75 |
| C | News articles, university news pages | 0.45 |
| D | Personal blogs, forums | 0.20 |

**Other components**, each in [0, 1]:
- *Recency:* exponential decay of (as_of − valid_from), with the half-life as a parameter.
- *Relevance:* the normalised reranker score.
- *Specificity:* how well the claim's scope matches the scope in the question.
- *IndependentSupport:* the number of **distinct publishers** supporting the cluster, after collapsing `derived_from` chains,
  so that copied text does not count as extra support.

**Weights are learned, not chosen:** a logistic regression on the dev split predicts "this value cluster is the gold
answer" from the five features. The coefficients become w1…w5 in §14, and the report lists them. We compare three settings:
(a) uniform weights, (b) authority only, (c) learned weights, and run a sensitivity analysis on the tier priors (±0.1).
This is how the formula is "experimentally justified".

### 4.7 Resolution and abstention (§16)

A deterministic procedure, applied to the value clusters:

1. **Scope.** Drop clusters whose scope does not match the question. If several scopes remain and the question does not
   pick one → `RESOLVED_CONTEXT` (answer per scope).
2. **Time.** If the question asks for the current value and a `CURRENT` cluster exists, `OUTDATED` and superseded clusters are
   set aside and reported as "previous" → `RESOLVED_TEMPORAL`.
3. **Score.** Cluster score = sum of SourceScore over its independent supporting sources. Let *m* be the margin between
   the top two clusters.
   - Only one cluster → `NO_CONFLICT`.
   - m ≥ τ and the top cluster has at least one source of tier B or higher → `RESOLVED_AUTHORITY` or `RESOLVED_MAJORITY`
     (whichever term dominates).
   - Otherwise → `UNRESOLVED`: give both values and their sources, and say so.
4. **Evidence.** If no claim is relevant enough → `INSUFFICIENT_EVIDENCE` (abstain).

τ is tuned on dev to balance answer accuracy against abstention precision.

### 4.8 Conflict types reported to the user (§9)

The pairwise labels (§4.3) are combined with the resolution step to give the conflict type the user sees, covering all six
categories in §9: `FACTUAL`, `NUMERICAL`, `TEMPORAL`, `VERSION` (temporal with an explicit same-lineage supersession),
`SCOPE`, `RELIABILITY` (a contradiction decided by an authority gap). The minimum product needs three (§31).

### 4.9 Citation-aware generation (§18)

- The LLM receives a **resolution brief**, not just raw passages. The brief contains the chosen value, the status, the
  supporting passages [S#], the superseded or conflicting passages with their labels and dates, and the conflict type. It is told to
  give the current value first, then mention the earlier or conflicting values with citations.
- **Citation check:** each answer sentence is checked with NLI against the passages it cites. Unsupported sentences are
  removed, and the event is logged (it feeds the faithfulness metric). This is a single verification pass, not the
  self-correction stretch goal.

### 4.10 Confidence (§17)

Two separate numbers, as the spec requires:

- **Consistency score:** the share of evidence weight that agrees with the chosen cluster, discounted by contradiction
  severity. It describes the evidence, not the answer.
- **Answer confidence:** a calibrated probability that the answer is correct. Logistic regression on dev, using these features:
  retrieval score, cluster margin, number of independent sources, maximum contradiction probability, temporal status, top
  authority tier, and citation-check failures; then isotonic calibration. Reported with ECE, Brier score and a reliability
  diagram. The LLM's own token probabilities are **not** used as confidence.

---

## 5. Systems compared (§23, §26)

| ID | System |
|---|---|
| B1 | Vanilla RAG: dense top-k, plain prompt |
| B2 | RAG + hybrid retrieval + reranking |
| B3 | B2 + a "conflict-aware" prompt only (tests §33's "prompt changes are not enough" directly) |
| V1 | B2 + NLI contradiction detection |
| V2 | B2 + LLM-only contradiction detection |
| **CR** | **Full ConsistRAG** |

Ablations of CR (§26): without source reliability · without temporal reasoning · without NLI · without the numeric module ·
without reranking · without the calibrated confidence (uncalibrated heuristic instead) · without independence dedup.

All systems use the same generator model, retrieval index, k, and decoding settings (temperature 0, fixed seed).

---

## 6. Benchmark (Deliverable 2)

### 6.1 End-to-end QA set

Target 300 items: ~150 on Corpus R and ~150 on Corpus S. Mix:

| Category | Share |
|---|---|
| Temporal / version | 25 % |
| Numerical | 15 % |
| Scope | 15 % |
| Source reliability | 15 % |
| Unresolvable (should report the conflict) | 10 % |
| No conflict (controls) | 15 % |
| Insufficient evidence (should abstain) | 5 % |

Item format (extends §22):

```json
{
  "id": "R-TMP-014",
  "corpus": "R",
  "split": "test",
  "question": "What is the current monthly grant for long-term study mobility in country group 1?",
  "as_of": "2026-10-01",
  "answer_type": "money",
  "gold_answer": {"amount": 0, "unit": "EUR", "per": "month"},
  "acceptable_answers": [],
  "gold_status": "RESOLVED_TEMPORAL",
  "conflict_type": "VERSION",
  "supporting_docs": ["PG2026"],
  "superseded_docs": ["PG2023"],
  "conflicting_docs": [],
  "gold_evidence": ["PG2026#c311"],
  "verified_by": null
}
```

Answers are short, typed values wherever possible, so correctness is scored automatically by normalised value match.
Open answers are only used where unavoidable.

**Splits:** 30 % dev (all tuning: k, θ, τ, weights, calibration) / 70 % test (touched only for final runs).

**Verification:** Claude drafts every item with an evidence pointer. The team then checks every test item against the
cited passage using a checklist (`docs/benchmark_guide.md`) and fills `verified_by`. Two people independently label a 50-item
subset; we report Cohen's κ.

### 6.2 Component sets

- **Claim pairs:** about 600 labelled pairs, drawn from both corpora and balanced over the §8 classes, for contradiction-detection
  metrics (§24.2).
- **Claim extraction gold:** 50 chunks with hand-written claims.
- **Retrieval qrels:** derived from `gold_evidence`.
- MNLI / ANLI / FEVER samples are used only to check that the NLI component is installed and behaving correctly (§21.3). They are not
  used for the end-to-end evaluation.

### 6.3 Robustness sets

Built from the test items:
- passage-order permutations, testing the order sensitivity described in §3
- distractor injection: extra low-authority documents with wrong values
- paraphrased questions

Metric: whether the answer stays the same, and accuracy under each perturbation.

---

## 7. Evaluation framework (Deliverable 7)

`python -m consistrag.eval run --config configs/cr.yaml --split test` writes per-item results to Postgres and
`results/<run_id>/` (CSV + JSON). The dashboard reads the same tables.

| Area | Metrics |
|---|---|
| Retrieval (§24.1) | Recall@k, Precision@k, MRR, nDCG@10 |
| Contradiction (§24.2) | Accuracy, per-class P/R/F1, macro-F1, confusion matrix |
| Correctness (§24.3) | Normalised value match (primary); LLM judge for open answers, with a strict rubric |
| Temporal | Outdated-answer rate (answered with a superseded value) |
| Resolution (§24.5) | Correctly resolved / resolvable; status accuracy |
| Abstention | Precision and recall of `UNRESOLVED` + `INSUFFICIENT_EVIDENCE` |
| Faithfulness (§24.4) | Citation precision and recall (NLI-checked, as in ALCE); share of supported sentences |
| Calibration (§24.6) | ECE, Brier score, reliability diagram; correlation of consistency score with correctness (§25 Q8) |
| Robustness | Answer stability under permutation, distractors and paraphrase |
| Cost (§25 Q5–6) | Latency p50/p95 per stage; prompt and completion tokens per query |

- **Judge:** a different local model family from the generator (e.g. generator Qwen2.5-7B, judge Llama-3.1-8B), so the generator
  never grades itself. The judge is checked against team labels on 50 items before its scores are used.
- **Significance:** paired bootstrap (10k resamples) for metric differences; McNemar's test for accuracy between two
  systems.
- **Error analysis (§27):** a script samples 30 failed test items, stratified by system and category. The team labels each
  with the §27 categories using the stage trace; we report counts and examples.

Each experimental question in §25 maps to a table or figure in the report: Q1 → B2 vs CR; Q2 → V1 vs V2 vs CR on the pair
set; Q3 → the without-source-reliability ablation; Q4 → outdated-answer rate, CR vs the without-temporal ablation; Q5–6 →
cost table; Q7 → abstention P/R; Q8 → calibration.

---

## 8. Implementation

### 8.1 Models (all local, all fit in 8 GB)

| Role | Model | Approx. VRAM |
|---|---|---|
| Generator, claim extractor, LLM classifier | Qwen2.5-7B-Instruct, Q4_K_M (Ollama) | ~5 GB |
| Judge (evaluation only) | Llama-3.1-8B-Instruct, Q4_K_M (Ollama) | ~5.5 GB, loaded separately |
| NLI | DeBERTa-v3-large (MNLI/FEVER/ANLI/LING/WANLI) fp16; base as fallback | ~0.9 GB |
| Embeddings | bge-base-en-v1.5 | ~0.3 GB |
| Reranker | bge-reranker-base | ~0.6 GB |

**CPU profile** (machines without an NVIDIA GPU): generator Qwen2.5-3B-Instruct Q4_K_M, NLI DeBERTa-v3-base, same embedder
and reranker, and the LLM step of the contradiction cascade (§4.3) switched off so a query stays under about a minute.
The profile is one config value; the pipeline code is the same. Evaluation results in the report come from the GPU
profile; the CPU profile is measured once for the cost table.

Model names, quantisation and Ollama model digests are recorded in every run's metadata. The LLM client speaks the
OpenAI-compatible API (Ollama provides one), so another backend can be plugged in without code changes.

### 8.2 Repository layout

```
backend/consistrag/
  config.py           pipeline configs (pydantic), loading configs/*.yaml
  models.py           shared pydantic types: Document, Chunk, Claim, Relation, Cluster, Resolution, Answer
  llm/                OpenAI-compatible client, JSON-schema calls, prompt templates
  ingest/             fetch, extract, metadata, chunk, embed, claims, normalise
  retrieval/          dense, bm25, fusion, rerank
  consistency/        pairing, rules, numeric, nli, llm_classifier, cascade, graph
  sources/            temporal, scoring
  resolution/         resolver
  generation/         brief, generator, citation_check
  confidence/         consistency_score, calibration
  pipeline.py         answer(question, config) — the only entry point the API, UI and eval use
  db/                 SQLAlchemy models, Alembic migrations
  api/                FastAPI app
  eval/               benchmark loader, runners, metrics, significance, error sampling, reports
backend/tests/        pytest (unit tests per module; LLM and NLI replaced by fakes)
frontend/             Next.js app
configs/              b1.yaml … cr.yaml, ablations/*.yaml
data/manifests/       corpus manifests (committed)
data/synth/           synthetic corpus generator + fact tables (committed)
data/benchmark/       QA items, claim pairs, extraction gold (committed)
data/raw/             fetched documents (gitignored)
results/              run outputs (gitignored)
docs/                 project_spec.md, design.md, literature.md, benchmark_guide.md, report/
docker-compose.yml    postgres+pgvector, ollama, api, web
```

### 8.3 API (§29)

`POST /query` · `POST /ingest` · `GET /sources` · `GET /documents/{id}` · `GET /claims/{id}` · `GET /conflicts` ·
`GET /evaluation` · `GET /runs/{id}/trace`. The `/query` response has the shape given in §29, plus `status`, `conflict_type`, the
clusters, and `trace_id`.

### 8.4 User interface (§19, §20)

- **Ask:**
  - the question and the answer with inline [S#] citations
  - the confidence and consistency scores
  - evidence cards with ✓ supports / ⚠ conflicts / ⏱ superseded badges, source tier and dates
  - the conflict type and resolution explanation
  - an expandable consistency-graph view
- **Sources:** the corpus browser with metadata.
- **Conflicts:** the stored claims and detected conflicts for each topic.
- **Evaluation:** a dashboard of runs, metric tables, the reliability diagram, and the ablation chart.

### 8.5 Engineering

- Python 3.11, `ruff` + `mypy`, pytest.
- GitHub Actions runs lint and unit tests with fakes for the LLM and NLI. CI has no GPU, and neither does the Claude cloud
  session.
- Full experiments run on the RTX 4060 machine with one command per experiment.
- Repeatability: temperature 0 and fixed seeds keep outputs stable on one machine. Outputs may differ slightly between
  machines (different GPU, CPU or model profile); that is accepted. LLM responses are cached by (model digest, prompt hash)
  so repeated evaluation runs are fast.

### 8.6 Deployment (Deliverable 8)

Target: a team member or grader with a Windows 10/11 PC follows `README.md` and has the UI open in the browser, with no
help. Outputs do not need to match ours exactly; the program needs to work.

**Prerequisites:** Docker Desktop (WSL2 backend; virtualisation enabled in BIOS). NVIDIA GPU optional; with one, a recent
driver is enough (Docker Desktop passes the GPU through WSL2). Minimum 8 GB RAM for the CPU profile, 16 GB recommended;
about 15 GB free disk.

**Services** (`docker-compose.yml`):

| Service | Image | Notes |
|---|---|---|
| `db` | `pgvector/pgvector:pg16` | On first start restores the prebuilt index (see below). |
| `ollama` | `ollama/ollama` | Models pulled on first start by a one-shot `ollama-init` service. |
| `api` | built from `backend/Dockerfile` | CPU PyTorch by default; build argument `TORCH=cuda` for the GPU profile. Small models (embedder, reranker, NLI) cached in a volume. |
| `web` | built from `frontend/Dockerfile` | Next.js production build. |

**Two ways to start:**
- `start.bat` — CPU profile, works everywhere.
- `start-gpu.bat` — adds `docker-compose.gpu.yml` (GPU access for `ollama` and `api`, GPU model profile).

Both scripts check that Docker is running and print the UI address when ready.

**Prebuilt index:** ingestion (claim extraction with the LLM) takes hours on a CPU, so no distribution machine runs it.
We ingest once on the RTX 4060, export a database dump (documents, chunks, embeddings, claims) and attach it to a GitHub
Release. `db` downloads and restores it on first start. The dump contains only data we may redistribute (EU documents
and the synthetic corpus); third-party pages are included only as extracted claims and short quoted passages with links.
`make ingest` (or `ingest.bat`) is still available for anyone who wants to rebuild it.

**First run:** downloads Docker images, models (CPU profile ~3 GB, GPU profile ~6 GB) and the index dump; internet is
needed once. An offline bundle (images saved with `docker save`, models and dump on a USB drive) is documented as a
fallback.

**Acceptance test:** before release, a fresh Windows machine **without** an NVIDIA GPU follows the README only, opens the
UI and answers the four §37 demo questions. A second run on the RTX 4060 with `start-gpu.bat`.

---

## 9. Phases

Ordered; each phase ends when its exit check passes. Deliverables D1–D9 are from §34.

| # | Phase | Output | Exit check |
|---|---|---|---|
| 0 | Skeleton | Repo layout, Docker Compose (db, ollama, api, web stubs), CPU and GPU start scripts, CI | `start.bat` brings up all services on Windows; `pytest` green in CI |
| 1 | Literature and requirements (D1) | `docs/literature.md`, `docs/requirements.md` (functional and non-functional requirements, incl. latency and hardware), conflict taxonomy | Every §34 D1 topic covered; requirements reviewed by the team |
| 2 | Baseline RAG + evaluation runner (D3, D7 start) | Ingestion, hybrid retrieval, rerank, B1/B2 generation with citations; `eval run` with retrieval and answer metrics | B1/B2 answer end to end on a seed synthetic corpus (first version of the generator); retrieval metrics computed |
| 3 | Benchmark (D2) | Corpus R manifest + fetch, synthetic generator, 300 QA items, pair set, extraction gold, robustness sets, `benchmark_guide.md` | Team has verified every test item; κ on the 50-item overlap reported |
| 4 | Claim extraction (D4 part) | Schema, extractor, normaliser, extraction evaluation | Extraction P/R measured on 50 gold chunks |
| 5 | Contradiction detection (D4 part) | Pairing, rules, numeric module, NLI, LLM classifier, cascade, pairwise labels | NLI vs LLM vs hybrid compared on the pair set |
| 6 | Source + temporal reasoning (D4 done, D5 part) | Validity intervals, supersession, consistency graph, source scoring, user-facing conflict types (§4.8) | All six conflict types produced; learned weights reported vs uniform / authority-only |
| 7 | Resolution and generation (D5 done) | Resolver, abstention, resolution brief, citation check, confidence | CR answers the four §37 demos correctly on the synthetic corpus |
| 8 | API + UI (D6) | FastAPI endpoints, Next.js pages incl. evaluation dashboard | All four §37 demos shown in the UI |
| 9 | Full evaluation (D7 done) | All systems and ablations on test, robustness, significance, calibration, cost, error analysis | All tables and figures regenerated by one script |
| 10 | Deployment (D8) | Prebuilt index release, final start scripts, README install guide, offline bundle notes | Acceptance test (§8.6) passes on a non-NVIDIA Windows machine |
| 11 | Report (D9) | 21 chapters per §34, figures from phase 9 | — |

Docker Compose and CI exist from Phase 0 and are kept working in every phase, so Phase 10 is packaging and testing, not
a first attempt.

### 9.1 Deliverable traceability

| Deliverable (§34) | Built in | Finished in | Evidence |
|---|---|---|---|
| D1 Literature and requirement analysis | 1 | 1 | `docs/literature.md`, `docs/requirements.md` |
| D2 Dataset and benchmark | 2–3 | 3 | `data/benchmark/`, `benchmark_guide.md`, verification record and κ |
| D3 Baseline RAG | 2 | 2 | B1/B2 configs and results |
| D4 Consistency analysis (extraction, detection, categorisation) | 4–6 | 6 | Extraction P/R, pair-set comparison, conflict-type output |
| D5 Conflict resolution (source ranking, temporal, resolution, abstention) | 6–7 | 7 | Weight study, demo answers, abstention metrics |
| D6 User interface | 8 | 8 | UI showing answer, evidence, conflicts, citations, confidence |
| D7 Evaluation framework | 2–9 | 9 | `consistrag.eval`, results tables, dashboard |
| D8 Deployment | 0–10 | 10 | Compose files, start scripts, README, acceptance-test record |
| D9 Final report | 1–11 | 11 | `docs/report/` |

### 9.2 Grading weights (§36) by phase

| Criterion | Weight | Phases |
|---|---:|---|
| Literature review and problem definition | 10 % | 1 |
| Baseline RAG quality | 10 % | 2 |
| Claim extraction | 10 % | 4 |
| Conflict detection | 15 % | 5 |
| Source / temporal reasoning | 15 % | 6 |
| Conflict-aware generation | 15 % | 7 |
| Evaluation methodology | 15 % | 2, 3, 9 |
| Software engineering and deployment | 5 % | 0, 8, 10 |
| Documentation and presentation | 5 % | 11 |

---

## 10. Literature to cover (Phase 1)

Each reference is checked before it is cited:

- **RAG:** Lewis et al. 2020; Self-RAG (Asai et al. 2023); Corrective RAG (Yan et al. 2024).
- **Knowledge conflicts:** survey by Xu et al. 2024; ConflictBank; WikiContradict; FreshQA (time-sensitive QA).
- **NLI and fact verification:** SNLI, MultiNLI, ANLI, FEVER, SciFact; DeBERTa.
- **Attribution and faithfulness:** ALCE (citation evaluation); RAGAS; FActScore.
- **Retrieval:** BM25; Reciprocal Rank Fusion (Cormack et al. 2009); cross-encoder reranking.
- **Calibration:** Guo et al. 2017 (ECE).

---

## 11. Risks

| Risk | Mitigation |
|---|---|
| A 7B model extracts claims poorly | JSON schema, few-shot examples, deterministic normaliser, sentence-level fallback; measured in Phase 3 before building on it |
| 8 GB VRAM | Only one LLM loaded at a time; NLI, embeddings and reranker are small; NLI drops to base size if needed |
| Machines without an NVIDIA GPU are slow | CPU profile: 3B model, NLI base, no LLM step in the cascade; prebuilt index so nothing is ingested on CPU |
| Docker Desktop problems on Windows (virtualisation off, WSL2 missing, WSL memory limit) | README troubleshooting section; start script checks Docker before starting; acceptance test on a clean machine |
| Large first-run download | Sizes stated in README; offline bundle documented |
| Slow queries | Claims stored at ingestion; candidate pairing (blocking); LLM classifier only for uncertain pairs; response cache |
| Programme Guides are long PDFs with tables | Section-aware chunking; tables kept whole; spot-check extraction on the grant-rate sections |
| Web pages without dates | Manifest dates set by hand; `UNKNOWN` temporal status handled explicitly |
| Benchmark written by the same assistant that builds the system | Labels created by the generator for Corpus S; evidence pointers + team verification + κ for all test items; tuning on dev only |
| LLM judge bias | Different model family; validated against team labels; primary metric is automatic value match |
| Third-party content licensing | Commit only manifests and fetch scripts; raw text stays local |
| Domain rejected | Backup domain (§1.1); only manifest, authority table and benchmark change |
