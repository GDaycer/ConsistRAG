# ConsistRAG — Schedule

The semester has 12 weeks. Work starts in **week 3** (Thursday 1 October 2026) and the project must be complete by
the **end of week 11** (Sunday 29 November 2026). Week 12 is kept free as buffer and for the presentation. Every task
is chained to the tasks it depends on. Red bars are the critical path; a delay there delays the end date. A rendered copy
is in `img/gantt.png`.

The chart axis shows semester weeks. Mermaid only understands calendar dates, so the chart uses a stand-in calendar
where week 1 starts on 1 January 2024; the table below gives the real dates. To shift the plan, change only the start
date of `p0`.

```mermaid
gantt
    title ConsistRAG — schedule, weeks 3–11 (week 12 buffer)
    dateFormat YYYY-MM-DD
    axisFormat Week %-W
    tickInterval 1week
    weekday monday
    todayMarker off

    section 0 Skeleton
    Repo layout, Docker Compose, CI                  :crit, p0, 2024-01-18, 3d

    section 1 Literature + requirements (D1)
    Literature review                                :p1a, after p0, 7d
    Requirements document                            :p1b, after p1a, 3d
    D1 done                                          :milestone, m1, after p1b, 0d

    section 2 Baseline RAG (D3)
    Ingestion, chunking, embeddings                  :crit, p2a, after p0, 4d
    Hybrid retrieval + reranking                     :crit, p2b, after p2a, 3d
    Generation with citations, eval runner           :crit, p2c, after p2b, 4d
    D3 done                                          :milestone, m3, after p2c, 0d

    section 3 Benchmark (D2)
    Synthetic corpus generator                       :p3a, after p0, 5d
    Erasmus+ manifest + fetch                        :p3b, after p3a, 6d
    Draft QA items, claim pairs, gold chunks         :p3c, after p3b, 7d
    Team verification of test items                  :p3d, after p3c, 7d
    D2 done                                          :milestone, m2, after p3d, 0d

    section 4 Claim extraction
    Extractor, normaliser, evaluation                :crit, p4, after p2c, 7d

    section 5 Contradiction detection
    Rules, numeric module, NLI, LLM, cascade         :crit, p5, after p4, 9d

    section 6 Source + temporal reasoning
    Temporal, graph, source scoring, types           :crit, p6, after p5, 9d
    D4 done                                          :milestone, m4, after p6, 0d

    section 7 Resolution + generation
    Resolver, abstention, citation check, confidence :crit, p7, after p6, 8d
    D5 done                                          :milestone, m5, after p7, 0d

    section 8 API + UI (D6)
    API endpoints                                    :p8a, after p2c, 4d
    UI pages                                         :p8b, after p5, 9d
    Integration with full pipeline                   :p8c, after p7, 3d
    D6 done                                          :milestone, m6, after p8c, 0d

    section 9 Full evaluation (D7)
    Systems, ablations, robustness, calibration      :crit, p9a, after p7 p3d, 6d
    Error analysis (30 cases)                        :crit, p9b, after p9a, 3d
    D7 done                                          :milestone, m7, after p9b, 0d

    section 10 Deployment (D8)
    Database dump release, README, pinning           :p10a, after p9a p8c, 3d
    Redeployment test on a fresh machine             :p10b, after p10a, 1d
    D8 done                                          :milestone, m8, after p10b, 0d

    section 11 Report (D9)
    Chapters 1–6 (intro to dataset design)           :p11a, after p1b, 10d
    Chapter 15 (experimental setup)                  :p11c, after p3d, 4d
    Chapters 7–14 (system chapters)                  :p11b, after p6, 14d
    Chapters 16–21 + final revision                  :crit, p11d, after p9b p10b, 3d
    D9 done                                          :milestone, m9, after p11d, 0d

    section Week 12
    Buffer, presentation                             :buf, after p11d, 7d
```

## Same plan with real dates

| Phase | Weeks | Start | End | Depends on |
|---|---|---|---|---|
| 0 Skeleton | 3 | Thu 1 Oct | Sat 3 Oct | — |
| 1 Literature + requirements (D1) | 3–5 | Sun 4 Oct | Tue 13 Oct | 0 |
| 2 Baseline RAG + eval runner (D3) | 3–5 | Sun 4 Oct | Wed 14 Oct | 0 |
| 3 Benchmark (D2) | 3–7 | Sun 4 Oct | Wed 28 Oct | 0 (team verification 22–28 Oct) |
| 4 Claim extraction | 5–6 | Thu 15 Oct | Wed 21 Oct | 2 |
| 5 Contradiction detection | 6–7 | Thu 22 Oct | Fri 30 Oct | 4 |
| 6 Source + temporal reasoning (D4) | 7–8 | Sat 31 Oct | Sun 8 Nov | 5 |
| 7 Resolution + generation (D5) | 9–10 | Mon 9 Nov | Mon 16 Nov | 6 |
| 8 API + UI (D6) | 5–10 | Thu 15 Oct | Thu 19 Nov | API after 2; UI after 5; integration after 7 |
| 9 Full evaluation (D7) | 10–11 | Tue 17 Nov | Wed 25 Nov | 7, team verification |
| 10 Deployment (D8) | 11 | Mon 23 Nov | Thu 26 Nov | 8, evaluation runs |
| 11 Report (D9) | 5–11 | Wed 14 Oct | Sun 29 Nov | chapters written after the phases they describe |
| Buffer, presentation | 12 | Mon 30 Nov | Sun 6 Dec | — |

## Notes

- **Compression:** the earlier plan needed about 14 weeks; this one fits in 8.5. More work runs in parallel (UI alongside
  source reasoning, report chapters alongside the system phases), and each phase is shorter. There is no slack before
  week 12.
- **Your tasks:** team verification (22–28 Oct), GPU runs on the RTX 4060 during phases 4–9 (claim extraction for the
  corpus, all evaluation runs 17–22 Nov), and the redeployment test (26 Nov).
- **Riskiest points:** claim extraction (one week) and the evaluation week, which can only start once phase 7 is done.
- **If a phase slips by more than two days,** cut scope in this order rather than moving the end date:
  1. benchmark 300 → 200 items and the pair set 600 → 400;
  2. robustness sets reduced to passage-order permutation only;
  3. Platt scaling instead of isotonic calibration;
  4. UI evaluation dashboard reduced to static result tables.
