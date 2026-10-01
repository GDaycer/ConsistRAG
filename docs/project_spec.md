> Original project brief, kept verbatim. Converted from the supplied `.txt` (encoding repaired; the conflict-graph sketch in section 15 was redrawn because its line breaks were lost). Our design decisions live in `design.md`.

# ConsistRAG
## Consistency-Aware Retrieval-Augmented Generation over Conflicting Knowledge Sources

**Project Type:** Senior Design / Graduation Project  
**Recommended Team Size:** 4–5 students  
**Primary Domains:** Retrieval-Augmented Generation (RAG), Large Language Models, Natural Language Processing, Information Retrieval, Contradiction Detection, Source Evaluation, Explainable AI  
**Target Output:** A working RAG system that can detect conflicting information across multiple sources, reason about source reliability and temporal validity, and generate citation-aware answers with explicit confidence and conflict explanations.

---

# 1. Project Vision

**ConsistRAG** aims to develop a Retrieval-Augmented Generation system that can identify, analyze, and manage contradictory information retrieved from multiple knowledge sources before producing a final answer.

Conventional RAG systems typically retrieve several relevant passages and pass them directly to a large language model. This approach works well when the retrieved information is mutually consistent. However, real-world knowledge bases frequently contain:

- outdated information,
- inconsistent policies,
- duplicate but non-identical statements,
- conflicting numerical values,
- different versions of the same document,
- source-specific interpretations,
- incorrect or low-quality content,
- temporally valid but superseded information.

A standard RAG pipeline may combine these conflicting passages and generate a misleading or internally inconsistent response.

The proposed system should therefore introduce an explicit **consistency analysis layer** between retrieval and generation.

The core idea is:

> **Retrieve → Extract Claims → Detect Conflicts → Evaluate Sources → Resolve or Explain Conflicts → Generate Answer with Evidence**

The project should produce a complete prototype and experimentally compare it against a standard RAG baseline.

---

# 2. Main Objectives

The project should achieve the following objectives:

1. Build a complete baseline RAG pipeline.
2. Retrieve evidence from multiple documents or knowledge sources.
3. Extract atomic factual claims from retrieved passages.
4. Detect semantic contradictions and inconsistencies between claims.
5. Distinguish direct contradiction from partial disagreement, outdated information, and missing context.
6. Evaluate evidence according to source quality, recency, authority, and relevance.
7. Resolve conflicts when sufficient evidence exists.
8. Explicitly report unresolved conflicts when no reliable resolution is possible.
9. Generate answers with source citations.
10. Produce a confidence or consistency score for each answer.
11. Evaluate the proposed approach against conventional RAG.
12. Provide an interactive user interface that shows retrieved evidence and detected conflicts.

---

# 3. Example Problem

Assume the following documents are retrieved.

### Source A

> The warranty period is two years.

### Source B

> The warranty period is three years.

### Source C

> Beginning in January 2026, the warranty period was increased from two years to three years.

A conventional RAG system may return either two years or three years depending on the retrieved context and prompt ordering.

ConsistRAG should instead identify that:

```text
Claim A: Warranty period = 2 years
Claim B: Warranty period = 3 years
Claim C: Warranty period changed from 2 to 3 years in January 2026
```

and reason that the apparent contradiction may be **temporal** rather than purely factual.

The final answer could contain:

```text
Current warranty period: 3 years.

The previous policy specified a 2-year warranty.
According to Source C, the policy changed in January 2026.

Confidence: High
Conflict Type: Temporal / Superseded Information
```

---

# 4. Expected System Architecture

A possible architecture is:

```text
                         USER QUESTION
                              │
                              ▼
                    ┌──────────────────┐
                    │ Query Processing │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Retrieval Layer  │
                    │ Vector / Hybrid  │
                    └────────┬─────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │ Retrieved Evidence   │
                  └──────────┬───────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Claim Extraction │
                    └────────┬─────────┘
                             │
                             ▼
              ┌────────────────────────────┐
              │ Consistency / Contradiction│
              │ Analysis Layer             │
              └─────────────┬──────────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Source Evaluation   │
                 │ + Temporal Analysis │
                 └──────────┬──────────┘
                            │
                            ▼
                ┌───────────────────────┐
                │ Conflict Resolution   │
                │ or Conflict Reporting │
                └───────────┬───────────┘
                            │
                            ▼
                  ┌────────────────────┐
                  │ Answer Generation  │
                  └─────────┬──────────┘
                            │
                            ▼
             ANSWER + CITATIONS + CONFIDENCE
```

---

# 5. Baseline RAG System

Before implementing consistency awareness, the team must first build a reliable baseline RAG pipeline.

The baseline should contain:

- Document ingestion
- Text extraction
- Chunking
- Embedding generation
- Vector database
- Similarity retrieval
- Optional keyword retrieval
- Reranking
- Context construction
- LLM-based answer generation
- Citation generation

Possible technologies:

- Python
- LangChain
- LlamaIndex
- Haystack
- Sentence Transformers
- Hugging Face
- OpenAI-compatible LLM APIs
- Local LLMs
- FAISS
- Chroma
- Qdrant
- Weaviate
- PostgreSQL + pgvector

The exact stack may be selected by the team.

---

# 6. Retrieval Strategies

The project should investigate more than one retrieval method.

Possible configurations:

## 6.1 Dense Retrieval

Embedding similarity:

```text
Question Embedding
        ↓
Vector Search
        ↓
Top-k Documents
```

---

## 6.2 Sparse Retrieval

Possible methods:

- BM25
- TF-IDF
- Keyword search

---

## 6.3 Hybrid Retrieval

Combine semantic and lexical retrieval:

```text
Dense Retrieval
      +
BM25 Retrieval
      ↓
Hybrid Ranking
```

Hybrid retrieval is strongly recommended.

---

## 6.4 Reranking

A reranker may be applied after initial retrieval.

Possible approaches:

- Cross-encoder reranker
- LLM reranker
- Reciprocal Rank Fusion
- Learned ranking model

---

# 7. Claim Extraction

One of the central components of the project is converting retrieved text into smaller factual units.

Example passage:

```text
The university introduced its new curriculum in 2025.
The curriculum contains 140 ECTS credits.
The previous curriculum contained 132 credits.
```

Extracted claims:

```text
C1: New curriculum introduced in 2025.
C2: Current curriculum contains 140 ECTS.
C3: Previous curriculum contained 132 ECTS.
```

Claim extraction can be implemented using:

- LLM prompting
- Sequence-to-sequence models
- Information extraction models
- Dependency parsing
- Open information extraction
- Structured output generation

Recommended representation:

```json
{
  "claim_id": "C102",
  "subject": "current curriculum",
  "predicate": "contains",
  "object": "140 ECTS",
  "source_id": "DOC12",
  "date": "2025",
  "confidence": 0.94
}
```

---

# 8. Contradiction Detection

The system must compare claims and determine whether they are mutually consistent.

Possible relationship classes include:

```text
ENTAILMENT
CONTRADICTION
PARTIAL_CONFLICT
TEMPORAL_CONFLICT
NUMERICAL_CONFLICT
CONTEXT_DEPENDENT
UNRELATED
UNKNOWN
```

At minimum, the system should support:

- contradiction,
- agreement,
- unrelated / insufficient evidence.

A stronger implementation should distinguish different conflict types.

---

# 9. Types of Conflicts

## 9.1 Direct Factual Conflict

```text
A: The program duration is 4 years.
B: The program duration is 5 years.
```

---

## 9.2 Numerical Conflict

```text
A: Tuition fee is 200,000 TRY.
B: Tuition fee is 250,000 TRY.
```

---

## 9.3 Temporal Conflict

```text
A: The dean is Person X.
B: The dean is Person Y.
```

Both may be correct at different dates.

---

## 9.4 Version Conflict

```text
Document v1: Minimum requirement is 60.
Document v2: Minimum requirement is 70.
```

The latest official version may supersede the old one.

---

## 9.5 Scope Conflict

```text
A: Attendance requirement is 70%.
B: Attendance requirement is 80%.
```

One statement may apply to undergraduate courses and the other to laboratory courses.

---

## 9.6 Source Reliability Conflict

```text
Official Regulation: 80%
Forum Post: 70%
```

The system should not necessarily treat all sources equally.

---

# 10. Contradiction Detection Methods

The team should compare multiple approaches.

Possible approaches include:

## Approach A — Natural Language Inference

Use an NLI model:

```text
Premise: The warranty period is 2 years.
Hypothesis: The warranty period is 3 years.

Result:
Contradiction = 0.96
```

Possible models:

- DeBERTa-based NLI
- RoBERTa-MNLI
- Modern transformer NLI models

---

## Approach B — LLM-Based Classification

Prompt an LLM to classify the relationship between two claims.

Expected structured output:

```json
{
  "relationship": "TEMPORAL_CONFLICT",
  "confidence": 0.91,
  "explanation": "The statements refer to different time periods."
}
```

---

## Approach C — Hybrid

Combine:

- rule-based checks,
- numerical comparison,
- NLI,
- LLM reasoning.

A hybrid design is strongly recommended.

---

# 11. Numerical Consistency Module

Numerical conflicts should preferably be handled explicitly rather than entirely through LLM reasoning.

Example:

```text
Claim A: Tuition = 240,000
Claim B: Tuition = 245,000
```

The system should identify:

```text
Difference = 5,000
Relative Difference = 2.08%
```

The design may support tolerance thresholds.

Example:

```text
if relative_difference < 1%:
    POSSIBLE_ROUNDING_DIFFERENCE
else:
    NUMERICAL_CONFLICT
```

---

# 12. Temporal Reasoning

Temporal reasoning should be an important part of the project.

Each source should ideally contain metadata such as:

```text
publication_date
effective_date
retrieval_date
version_date
last_updated
```

The system should attempt to distinguish:

```text
OLD INFORMATION
CURRENT INFORMATION
FUTURE INFORMATION
UNKNOWN TEMPORAL STATUS
```

Example:

```text
2024 regulation → 70%
2026 regulation → 80%
```

If the user asks:

> What is the current attendance requirement?

the 2026 regulation should normally be prioritized.

---

# 13. Source Reliability Model

Not all sources should have equal weight.

Possible source dimensions:

- Authority
- Recency
- Directness
- Specificity
- Evidence quality
- Agreement with independent sources

Example categories:

```text
Official Regulation      → Very High Authority
Official University Page → High Authority
News Article              → Medium
Personal Blog             → Low
Forum Post                → Very Low
```

The scoring method must be transparent and documented.

The project should avoid creating arbitrary weights without justification.

---

# 14. Example Source Score

An illustrative formula could be:

```text
SourceScore =
    w1 × Authority
  + w2 × Recency
  + w3 × Relevance
  + w4 × Specificity
  + w5 × IndependentSupport
```

The exact formula must be experimentally justified.

---

# 15. Conflict Graph

A strong implementation may represent evidence as a graph.

Example:

```text
            Claim A
           /       \
     SUPPORT       CONTRADICT
         /           \
    Claim C -------- Claim B
```

Possible nodes:

- claims
- documents
- sources
- entities

Possible edges:

- supports
- contradicts
- supersedes
- derived-from
- same-as
- temporally-precedes

This can form a **Consistency Graph**.

---

# 16. Conflict Resolution

The system should not always force a single answer.

Possible strategies:

## Resolution Type 1 — Clear Winner

Example:

```text
Official 2026 regulation supersedes 2024 regulation.
```

Result:

```text
Resolved
```

---

## Resolution Type 2 — Majority Evidence

Example:

```text
4 independent reliable sources support Claim A.
1 weak source supports Claim B.
```

Result:

```text
Claim A is more strongly supported.
```

---

## Resolution Type 3 — Contextual Resolution

Example:

```text
Source A applies to undergraduate students.
Source B applies to graduate students.
```

Result:

```text
No real contradiction after context separation.
```

---

## Resolution Type 4 — Unresolved

Example:

```text
Two equally authoritative current sources disagree.
```

The system should explicitly state:

```text
The available sources conflict and the system cannot confidently determine a single answer.
```

This is preferable to hallucinating certainty.

---

# 17. Confidence Estimation

Each answer should include an internal or visible confidence score.

Confidence may depend on:

- Retrieval relevance
- Number of supporting sources
- Source reliability
- Contradiction severity
- Temporal consistency
- Agreement between sources

Example:

```text
Answer Confidence: 0.87
Consistency Score: 0.92
Supporting Sources: 4
Conflicting Sources: 1
```

The system should distinguish **model confidence** from actual correctness.

---

# 18. Citation-Aware Answer Generation

Every factual answer should ideally provide evidence.

Example:

```text
The current warranty period is three years [S2, S4].

An older document states two years [S1], but a 2026 policy update
explicitly states that the period was increased to three years [S4].
```

The interface should allow the user to inspect the underlying source passages.

---

# 19. User Interface

A web-based interface is expected.

The interface should display:

- User question
- Final answer
- Confidence score
- Consistency score
- Retrieved documents
- Supporting claims
- Conflicting claims
- Source metadata
- Conflict type
- Resolution explanation
- Citations

---

# 20. Example UI Layout

```text
-------------------------------------------------
Question
-------------------------------------------------

What is the current warranty period?

-------------------------------------------------
Answer
-------------------------------------------------

The current warranty period is 3 years.

Confidence: 91%
Consistency: 88%

-------------------------------------------------
Evidence
-------------------------------------------------

✓ Source A — supports 3 years
✓ Source B — supports 3 years
⚠ Source C — states 2 years

-------------------------------------------------
Conflict Explanation
-------------------------------------------------

Source C is older and appears to describe the
previous policy.

-------------------------------------------------
```

---

# 21. Dataset Requirements

The project requires a dataset containing meaningful conflicts.

The team may use one or more of the following approaches.

---

## 21.1 Synthetic Conflict Dataset

Create controlled versions of documents containing:

- modified dates,
- changed numerical values,
- contradictory policies,
- superseded versions,
- partially conflicting statements.

This allows precise ground truth.

---

## 21.2 Public Multi-Source Dataset

Use public documents from domains where multiple versions exist.

Possible domains:

- university regulations,
- product documentation,
- public policies,
- technical manuals,
- software documentation,
- institutional procedures.

---

## 21.3 Existing NLI / Fact Verification Data

Existing datasets may be used for individual components.

Examples may include:

- SNLI
- MultiNLI
- FEVER
- ANLI
- SciFact

These should not replace the project-specific end-to-end evaluation dataset.

---

# 22. Ground Truth

Evaluation requires a manually verified test set.

Each test case should contain:

```text
Question
Relevant documents
Claims
Conflict type
Correct resolution
Expected answer
Expected evidence
```

Example:

```json
{
  "question": "What is the current warranty period?",
  "conflict_type": "TEMPORAL",
  "correct_answer": "3 years",
  "supporting_sources": ["S2", "S3"],
  "superseded_sources": ["S1"]
}
```

---

# 23. Evaluation Framework

The project must include a rigorous experimental evaluation.

At minimum compare:

```text
Baseline RAG
vs.
ConsistRAG
```

Additional variants are encouraged.

Example:

```text
RAG
RAG + Reranking
RAG + NLI
RAG + LLM Conflict Detection
ConsistRAG Full Pipeline
```

---

# 24. Evaluation Metrics

## 24.1 Retrieval Metrics

Possible metrics:

- Recall@K
- Precision@K
- MRR
- nDCG

---

## 24.2 Contradiction Detection Metrics

- Accuracy
- Precision
- Recall
- F1 Score
- Macro F1

---

## 24.3 Answer Correctness

Possible evaluation:

- Exact match
- Semantic similarity
- LLM-as-a-judge with strict rubric
- Human evaluation

---

## 24.4 Faithfulness

Measure whether the generated answer is supported by retrieved evidence.

---

## 24.5 Conflict Resolution Accuracy

Example:

```text
Correctly resolved conflicts
-----------------------------
Total resolvable conflicts
```

---

## 24.6 Uncertainty Calibration

The system should ideally assign lower confidence when evidence is conflicting.

Possible measures:

- Expected Calibration Error
- Brier Score

Optional but valuable.

---

# 25. Experimental Questions

The final report should answer questions such as:

1. Does explicit contradiction detection improve RAG factual accuracy?
2. Which contradiction detection approach works best?
3. Does source reliability weighting improve conflict resolution?
4. Does temporal reasoning reduce outdated answers?
5. How does ConsistRAG affect latency?
6. What is the token-cost overhead?
7. Does the system correctly abstain when a conflict cannot be resolved?
8. Does the consistency score correlate with actual answer correctness?

---

# 26. Ablation Study

A strong project should include an ablation analysis.

Example:

```text
Full ConsistRAG
- without source reliability
- without temporal reasoning
- without NLI
- without reranking
- without confidence estimation
```

This helps determine which component provides the most value.

---

# 27. Error Analysis

The team must analyze failed cases.

Possible categories:

```text
Retrieval failure
Claim extraction failure
Entity mismatch
Temporal reasoning failure
Numerical parsing failure
Incorrect contradiction classification
Incorrect source weighting
Generation hallucination
Citation mismatch
```

At least 20–30 representative errors should be manually inspected in the final evaluation.

---

# 28. Recommended Technology Stack

Possible stack:

## Language

- Python

## RAG

- LangChain
- LlamaIndex
- Haystack

## Embeddings

- Sentence Transformers
- BGE
- E5
- other modern embedding models

## Vector Database

- FAISS
- Qdrant
- Chroma
- pgvector
- Weaviate

## NLI

- Hugging Face Transformers
- DeBERTa / RoBERTa NLI models

## Backend

- FastAPI

## Frontend

- React
- Next.js
- Streamlit for prototype only

## Database

- PostgreSQL

## Deployment

- Docker
- Docker Compose

---

# 29. API Design

Suggested endpoints:

```text
POST /query
GET  /sources
GET  /documents/{id}
GET  /claims/{id}
GET  /conflicts
GET  /evaluation
```

Example response:

```json
{
  "answer": "The current warranty period is 3 years.",
  "confidence": 0.91,
  "consistency_score": 0.88,
  "sources": ["S2", "S4"],
  "conflicts": [
    {
      "source": "S1",
      "type": "TEMPORAL_CONFLICT",
      "status": "SUPERSEDED"
    }
  ]
}
```

---

# 30. Suggested Team Distribution

For a 5-person team:

## Student 1 — Retrieval & RAG Lead

Responsibilities:

- document ingestion,
- chunking,
- embeddings,
- vector search,
- hybrid retrieval,
- reranking.

---

## Student 2 — Claim Extraction & NLP Lead

Responsibilities:

- claim decomposition,
- entity normalization,
- structured extraction,
- claim representation.

---

## Student 3 — Consistency & Conflict Detection Lead

Responsibilities:

- NLI,
- contradiction classification,
- numerical conflict detection,
- conflict graph.

---

## Student 4 — Source Evaluation & Answer Generation Lead

Responsibilities:

- source scoring,
- temporal reasoning,
- conflict resolution,
- confidence estimation,
- citation-aware generation.

---

## Student 5 — Backend, UI & Evaluation Lead

Responsibilities:

- API,
- frontend,
- experiment framework,
- dashboards,
- automated evaluation,
- deployment.

---

# 31. Minimum Viable Product

The minimum acceptable system should include:

- Baseline RAG
- Multiple-source retrieval
- Claim extraction
- Contradiction detection
- At least three conflict categories
- Source metadata
- Conflict-aware answer generation
- Citations
- Confidence score
- Web interface
- Baseline comparison
- Quantitative evaluation

---

# 32. Full Target System

A strong project should additionally include:

- Hybrid retrieval
- Reranking
- Temporal reasoning
- Numerical conflict detection
- Source authority scoring
- Conflict graph
- Abstention when evidence is unresolved
- Confidence calibration
- Detailed evaluation dashboard
- Ablation study
- Docker deployment
- Automated tests

---

# 33. What Will NOT Be Considered Sufficient

The following will not be enough:

- a standard PDF chatbot,
- a RAG system with only prompt changes,
- asking an LLM “are these documents contradictory?” without a systematic pipeline,
- no ground-truth dataset,
- no quantitative comparison,
- no baseline,
- no source metadata,
- no citations,
- no explicit conflict categories,
- no evaluation,
- only a Streamlit demo without substantial backend/NLP work.

The project must contain a genuine **consistency reasoning architecture**.

---

# 34. Expected Deliverables

## Deliverable 1 — Literature and Requirement Analysis

Document:

- RAG architecture,
- contradiction detection,
- NLI,
- fact verification,
- temporal reasoning,
- source reliability,
- relevant datasets.

---

## Deliverable 2 — Dataset and Benchmark

A project-specific benchmark containing:

- questions,
- documents,
- conflicting claims,
- conflict labels,
- correct answers,
- evidence labels.

---

## Deliverable 3 — Baseline RAG

A working standard RAG implementation.

---

## Deliverable 4 — Consistency Analysis Module

Working:

- claim extraction,
- contradiction detection,
- conflict categorization.

---

## Deliverable 5 — Conflict Resolution Module

Working:

- source ranking,
- temporal analysis,
- resolution logic,
- abstention logic.

---

## Deliverable 6 — User Interface

Interactive interface showing:

- answer,
- evidence,
- conflict details,
- citations,
- confidence.

---

## Deliverable 7 — Evaluation Framework

Automated experiments comparing baseline and proposed system.

---

## Deliverable 8 — Deployment

Dockerized or otherwise reproducible deployment.

---

## Deliverable 9 — Final Technical Report

Suggested chapters:

1. Introduction
2. Problem Definition
3. Related Work
4. RAG Background
5. System Requirements
6. Dataset Design
7. Retrieval Architecture
8. Claim Extraction
9. Consistency Analysis
10. Source Reliability
11. Temporal Reasoning
12. Conflict Resolution
13. Answer Generation
14. Implementation
15. Experimental Setup
16. Results
17. Ablation Study
18. Error Analysis
19. Limitations
20. Future Work
21. Conclusion

---

# 35. Suggested Semester Milestones

## Phase 1 — Literature Review and Dataset Design

- Study RAG and consistency problems.
- Select domain.
- Define conflict taxonomy.
- Build initial benchmark.

**Output:** Requirements + benchmark specification

---

## Phase 2 — Baseline RAG

- Document ingestion
- Embeddings
- Retrieval
- Baseline answer generation

**Output:** Working baseline

---

## Phase 3 — Claim Extraction

- Define claim schema
- Implement extraction
- Evaluate extraction quality

**Output:** Structured evidence representation

---

## Phase 4 — Contradiction Detection

- Implement NLI baseline
- Implement LLM-based classifier
- Compare alternatives

**Output:** Conflict detection module

---

## Phase 5 — Source and Temporal Reasoning

- Add source metadata
- Add reliability logic
- Add temporal conflict handling

**Output:** Consistency reasoning layer

---

## Phase 6 — Conflict-Aware Generation

- Implement resolution
- Implement abstention
- Add citations and confidence

**Output:** Full ConsistRAG pipeline

---

## Phase 7 — UI and API

- Backend service
- Visualization
- Evidence inspection
- Conflict explanation

**Output:** Integrated prototype

---

## Phase 8 — Evaluation

- Baseline comparison
- Ablation
- Error analysis
- Performance analysis

**Output:** Final experimental results

---

# 36. Suggested Evaluation Criteria

| Component | Weight |
|---|---:|
| Literature review and problem definition | 10% |
| Baseline RAG quality | 10% |
| Claim extraction | 10% |
| Conflict detection | 15% |
| Source / temporal reasoning | 15% |
| Conflict-aware generation | 15% |
| Evaluation methodology | 15% |
| Software engineering and deployment | 5% |
| Documentation and presentation | 5% |

---

# 37. Example Final Demo Scenarios

The final demonstration should include multiple conflict types.

### Scenario 1 — Current Policy

> What is the current attendance requirement?

System should identify old and new regulations.

---

### Scenario 2 — Numerical Conflict

> What is the annual tuition fee?

System should detect different numbers and explain date/source differences.

---

### Scenario 3 — Source Authority

> What is the official program duration?

System should prioritize official documentation over informal sources.

---

### Scenario 4 — Unresolved Conflict

> What is the exact eligibility threshold?

If two equally authoritative current sources conflict, the system should explicitly report uncertainty.

---

# 38. Stretch Goals

Optional extensions include:

## Multi-Agent Consistency Checking

Separate agents:

```text
Retriever Agent
Claim Extraction Agent
Fact Checker Agent
Temporal Reasoning Agent
Source Critic Agent
Answer Agent
```

---

## Graph-Based Consistency Reasoning

Build a graph containing:

```text
Documents
Claims
Entities
Sources
Support Edges
Conflict Edges
Temporal Edges
```

---

## Cross-Document Entity Resolution

Detect that:

```text
Istanbul Kultur University
IKU
İstanbul Kültür Üniversitesi
```

refer to the same entity.

---

## Self-Correction

After drafting the answer, the system performs a final consistency check:

```text
Draft Answer
     ↓
Evidence Verification
     ↓
Corrected Final Answer
```

---

# 39. Final Project Expectation

At the end of the semester, ConsistRAG should demonstrate that a RAG system can do more than simply retrieve documents and ask an LLM to generate an answer.

The final system should explicitly reason about:

- whether sources agree,
- why they disagree,
- which source is more reliable,
- whether the disagreement is temporal,
- whether a single answer can be justified,
- and when the system should admit uncertainty.

The expected contribution is therefore a complete:

> **Retrieval + Claim Extraction + Contradiction Detection + Source Evaluation + Temporal Reasoning + Conflict Resolution + Citation-Aware Generation**

pipeline.

---

# 40. Project Success Definition

The project will be considered successful when the team demonstrates that:

1. A standard RAG baseline is implemented.
2. Conflicting evidence is automatically detected.
3. Multiple conflict types are supported.
4. Source quality and recency are considered.
5. Temporal conflicts can be distinguished from direct contradictions.
6. The system can resolve some conflicts automatically.
7. Unresolved cases are explicitly reported.
8. Answers are supported with citations.
9. Confidence / consistency indicators are generated.
10. ConsistRAG is quantitatively compared with baseline RAG.
11. The system is reproducible and deployable.
12. The final prototype demonstrates measurable improvement in consistency and factual reliability.

---

## Project Name

# **ConsistRAG**

### Suggested Subtitle

**Consistency-Aware Retrieval-Augmented Generation over Conflicting Knowledge Sources**