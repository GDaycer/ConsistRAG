# ConsistRAG — Schedule

Baseline plan for the phases in `design.md` §9. There are no fixed deadlines, so dates are estimates from a start on
**Monday 5 October 2026**. Every task is chained to the tasks it depends on: to shift the whole plan, change only the
start date of `p0` below. Red bars are the critical path; a delay there delays the end date. A rendered copy is in `img/gantt.png`.

```mermaid
gantt
    title ConsistRAG — baseline schedule (about 14 weeks)
    dateFormat YYYY-MM-DD
    axisFormat %d %b
    tickInterval 1week
    todayMarker off

    section 0 Skeleton
    Repo layout, Docker Compose, CI               :crit, p0, 2026-10-05, 4d

    section 1 Literature + requirements (D1)
    Literature review                             :p1a, after p0, 7d
    Requirements document                         :p1b, after p1a, 3d
    D1 done                                       :milestone, m1, after p1b, 0d

    section 2 Baseline RAG (D3)
    Ingestion, chunking, embeddings               :crit, p2a, after p0, 5d
    Hybrid retrieval + reranking                  :crit, p2b, after p2a, 4d
    Generation with citations, eval runner        :crit, p2c, after p2b, 5d
    D3 done                                       :milestone, m3, after p2c, 0d

    section 3 Benchmark (D2)
    Synthetic corpus generator                    :p3a, after p0, 7d
    Erasmus+ manifest + fetch                     :p3b, after p3a, 7d
    Draft QA items, claim pairs, gold chunks      :p3c, after p3b, 10d
    Team verification of test items               :p3d, after p3c, 7d
    D2 done                                       :milestone, m2, after p3d, 0d

    section 4 Claim extraction
    Extractor, normaliser, evaluation             :crit, p4, after p2c, 10d

    section 5 Contradiction detection
    Rules, numeric module, NLI, LLM, cascade      :crit, p5, after p4, 14d

    section 6 Source + temporal reasoning
    Temporal, graph, source scoring, types        :crit, p6, after p5, 14d
    D4 done                                       :milestone, m4, after p6, 0d

    section 7 Resolution + generation
    Resolver, abstention, citation check, confidence :crit, p7, after p6, 14d
    D5 done                                       :milestone, m5, after p7, 0d

    section 8 API + UI (D6)
    API endpoints                                 :p8a, after p2c, 5d
    UI pages                                      :p8b, after p6, 10d
    Integration with full pipeline                :p8c, after p7, 4d
    D6 done                                       :milestone, m6, after p8c, 0d

    section 9 Full evaluation (D7)
    Systems, ablations, robustness, calibration   :crit, p9a, after p7 p3d, 10d
    Error analysis (30 cases)                     :crit, p9b, after p9a, 5d
    D7 done                                       :milestone, m7, after p9b, 0d

    section 10 Deployment (D8)
    Database dump release, README, pinning        :crit, p10a, after p9a p8c, 5d
    Redeployment test on a fresh machine          :crit, p10b, after p10a, 2d
    D8 done                                       :milestone, m8, after p10b, 0d

    section 11 Report (D9)
    Chapters 1–6 (intro to dataset design)        :p11a, after p1b, 10d
    Chapters 7–14 (system chapters)               :p11b, after p6, 14d
    Chapters 15–21 + final revision               :crit, p11c, after p9b p10b, 14d
    D9 done                                       :milestone, m9, after p11c, 0d
```

## Same plan as a table

| Phase | Start | End | Depends on |
|---|---|---|---|
| 0 Skeleton | 5 Oct | 8 Oct | — |
| 1 Literature + requirements (D1) | 9 Oct | 18 Oct | 0 |
| 2 Baseline RAG + eval runner (D3) | 9 Oct | 22 Oct | 0 |
| 3 Benchmark (D2) | 9 Oct | 8 Nov | 0 (team verification is the last week) |
| 4 Claim extraction | 23 Oct | 1 Nov | 2 |
| 5 Contradiction detection | 2 Nov | 15 Nov | 4 |
| 6 Source + temporal reasoning (D4) | 16 Nov | 29 Nov | 5 |
| 7 Resolution + generation (D5) | 30 Nov | 13 Dec | 6 |
| 8 API + UI (D6) | 23 Oct | 17 Dec | API after 2; UI after 6; integration after 7 |
| 9 Full evaluation (D7) | 14 Dec | 28 Dec | 7, team verification |
| 10 Deployment (D8) | 24 Dec | 30 Dec | 8, 9 |
| 11 Report (D9) | 19 Oct | 13 Jan | chapters written after the phases they describe |

## Notes

- **Your tasks:** the team verification week (phase 3), running GPU experiments on the RTX 4060 (phases 4–9), and the
  redeployment test (phase 10). Everything else Claude can draft or build, with your review.
- **Buffer:** the plan has no slack built in. The end-of-year holidays fall in phases 9–10; if the semester ends later,
  move them out rather than compressing the evaluation.
- **Riskiest estimate:** claim extraction (phase 4). If a 7B model extracts claims poorly, the fallback in `design.md`
  §4.1 is used and phase 5 starts on time.
