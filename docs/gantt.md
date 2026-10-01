# ConsistRAG — Schedule

An indicative plan of the deliverables in `project_spec.md` §34 over the 12-week semester. Work runs from week 3 to
week 11; week 12 is for the presentation. The detailed phase plan is in `design.md` §9. A rendered copy is in
`img/gantt.png`.

```mermaid
gantt
    title ConsistRAG — deliverables by semester week
    dateFormat YYYY-MM-DD
    axisFormat Week %-W
    tickInterval 1week
    weekday monday
    todayMarker off

    section Deliverables
    D1 Literature and requirement analysis   :d1, 2024-01-15, 14d
    D2 Dataset and benchmark                 :d2, 2024-01-22, 14d
    D3 Baseline RAG                          :d3, 2024-01-29, 14d
    D4 Consistency analysis module           :d4, 2024-02-05, 14d
    D5 Conflict resolution module            :d5, 2024-02-12, 14d
    D6 User interface                        :d6, 2024-02-19, 7d
    D7 Evaluation framework                  :d7, 2024-02-26, 7d
    D8 Deployment                            :d8, 2024-03-04, 14d
    D9 Final technical report                :d9, 2024-01-22, 56d
    Project complete                         :milestone, fin, 2024-03-18, 0d

    section Week 12
    Presentation                             :p, 2024-03-18, 7d
```

| Deliverable | Weeks |
|---|---|
| D1 Literature and requirement analysis | 3–4 |
| D2 Dataset and benchmark | 4–5 |
| D3 Baseline RAG | 5–6 |
| D4 Consistency analysis module | 6–7 |
| D5 Conflict resolution module | 7–8 |
| D6 User interface | 8 |
| D7 Evaluation framework | 9 |
| D8 Deployment | 10–11 |
| D9 Final technical report | 4–11 |
| Presentation | 12 |
