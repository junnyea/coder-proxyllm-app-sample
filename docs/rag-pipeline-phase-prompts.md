# RAG Pipeline — Phase Prompts

**Status:** TEMPLATE — not yet ready for planning.
Every `[BRACKETED]` value below must be replaced with a decided value before
`phase-planner` may plan Phase 1. The planner refuses to plan while any remain.

This file is the source of truth for scope, requirements, and Definitions of
Done. The orchestration contract (who does what, in what order) lives in
`CLAUDE.md` and does not repeat scope. Where the two ever conflict, this file
wins on *what* and `CLAUDE.md` wins on *how*.

---

## 0. Shared context (fill in once, binds all three phases)

| Decision | Value |
|---|---|
| Use case / question domain | `[USE_CASE]` |
| Primary users | `[USERS]` |
| Corpus source & location | `[CORPUS_SOURCE]` |
| Corpus size (docs / tokens) | `[CORPUS_SIZE]` |
| Document formats | `[FORMATS]` |
| Update cadence (static / batch / streaming) | `[UPDATE_CADENCE]` |
| Embedding model | `[EMBED_MODEL]` |
| Generation model | `[GEN_MODEL]` |
| Vector store | `[VECTOR_STORE]` |
| Language(s) | `[LANGUAGES]` |
| Deployment target | `[DEPLOY_TARGET]` |
| Data sensitivity / compliance regime | `[SENSITIVITY]` |
| Cost ceiling per 1k queries | `[COST_CEILING]` |
| Human owner (approves gates) | `[OWNER]` |

### Repository layout (fixed across phases)
```
config/          all tunables — no magic numbers in code
src/ingest/      loaders, cleaners, chunkers
src/index/       embedding + vector store adapters
src/retrieve/    retrieval strategies
src/generate/    prompt assembly + grounded answering
eval/            golden_dataset.jsonl, harness, reports
plans/           phase-N-plan.md
docs/            this file, runbook
tests/           unit + integration
```

### Global constraints
- Secrets from environment / secret manager only. Never hardcoded, logged, or committed.
- Retrieved documents are **untrusted input**; generation prompts must resist
  instructions embedded in corpus content.
- Every tunable (chunk size, overlap, `k`, model IDs, thresholds, timeouts,
  prompt templates) is config-declared and overridable per environment.
- No phase may implement a later phase's features. Scope creep is a Blocking
  review finding.

---

## Phase 1 — Classic RAG baseline

**Goal.** A working, measured, end-to-end RAG pipeline that answers
`[USE_CASE]` questions grounded in `[CORPUS_SOURCE]`, plus the frozen
evaluation apparatus that every later phase is judged against.

**Owner agents.** `pipeline-engineer` (pipeline), `eval-engineer` (dataset +
harness). Both review through `code-reviewer`.

### In scope
1. **Ingestion** — load `[FORMATS]` from `[CORPUS_SOURCE]`; normalize text;
   strip boilerplate; preserve per-document metadata (`source_id`, `title`,
   `url_or_path`, `ingested_at`, plus `[EXTRA_METADATA]`). Deterministic and
   idempotent: re-running over an unchanged corpus produces an identical index.
2. **Chunking** — strategy `[CHUNK_STRATEGY]` (e.g. recursive-character,
   sentence-window, structural/heading-aware). Target size `[CHUNK_SIZE]`
   tokens, overlap `[CHUNK_OVERLAP]`. Every chunk carries its parent document
   metadata and a stable chunk ID.
3. **Indexing** — embed with `[EMBED_MODEL]`; persist to `[VECTOR_STORE]`.
   Batched, resumable, rate-limit aware. Index build is a repeatable command.
4. **Retrieval** — dense top-`[K]` similarity search with metadata filtering.
   Single-stage only; **no reranking in Phase 1**.
5. **Generation** — assemble retrieved context into a grounded prompt for
   `[GEN_MODEL]`. Must cite sources per claim, must refuse when context is
   insufficient rather than answering from parametric memory, and must treat
   chunk text as data rather than instructions.
6. **Golden dataset** — `eval/golden_dataset.jsonl`, `[N_GOLDEN]` questions
   spanning `[QUESTION_MIX]` (e.g. factual lookup, multi-hop, comparison,
   unanswerable). Each record: `question`, `ground_truth_answer`,
   `relevant_chunk_ids`, `question_type`, `difficulty`.
7. **Eval harness** — reproducible run producing retrieval metrics
   (recall@k, MRR, nDCG@k), generation metrics (faithfulness /
   groundedness, answer relevance, citation accuracy), and latency +
   cost per query. Judge model `[JUDGE_MODEL]`, judge prompts versioned in
   config. Output: a dated report under `eval/reports/`.

### Explicitly out of scope
Reranking, hybrid/sparse retrieval, query rewriting, multi-turn conversation,
HTTP service layer, authentication, autoscaling, CI/CD.

### Definition of Done — Phase 1
- [ ] `make ingest && make index && make query` (or documented equivalents) run clean from a bare checkout.
- [ ] Recall@`[K]` ≥ `[P1_RECALL_TARGET]` on the golden dataset.
- [ ] Faithfulness ≥ `[P1_FAITHFULNESS_TARGET]`; citation accuracy ≥ `[P1_CITATION_TARGET]`.
- [ ] Unanswerable questions are refused ≥ `[P1_REFUSAL_TARGET]` of the time.
- [ ] p95 end-to-end latency ≤ `[P1_LATENCY_TARGET]`.
- [ ] Baseline report committed at `eval/reports/phase-1-baseline.md`.
- [ ] Zero secrets in the diff; all tunables in `config/`.
- [ ] `code-reviewer` Blocking findings: none outstanding.
- [ ] **`[OWNER]` signs off — at which point `eval/golden_dataset.jsonl` is FROZEN.**

> After sign-off the golden dataset is immutable. Any later change needs written
> approval from `[OWNER]`, recorded in the relevant `plans/phase-N-plan.md`.

---

## Phase 2 — Cohere rerank (two-stage retrieval)

**Goal.** Raise retrieval precision by over-retrieving then reranking, proven by
an honest A/B against the frozen Phase 1 baseline.

**Owner agents.** `rerank-engineer` (implementation), `eval-engineer` (A/B).

### In scope
1. **Over-retrieval** — widen stage-1 dense retrieval to `[CANDIDATE_K]`
   candidates (from `[K]`).
2. **Rerank** — `[COHERE_RERANK_MODEL]` scores candidates; keep top `[FINAL_K]`
   above relevance threshold `[RELEVANCE_THRESHOLD]`. If nothing clears the
   threshold, the pipeline reports insufficient context rather than padding.
3. **Fallback & resilience** — timeout `[RERANK_TIMEOUT]`, `[RERANK_RETRIES]`
   retries with backoff. On failure, degrade gracefully to Phase 1 dense-only
   results and emit a structured warning. A Cohere outage must never fail a query.
4. **Flag gating** — `[RERANK_FLAG]` in config toggles reranking at runtime,
   defaulting to `[RERANK_DEFAULT]`. Both paths stay exercised by tests.
5. **A/B evaluation** — same frozen dataset, same judge, rerank on vs. off.
   Report deltas on every Phase 1 metric plus added latency and Cohere cost.
6. **Optional experiment** — swap `[EMBED_MODEL]` for `[COHERE_EMBED_MODEL]` at
   stage 1 and measure. Ships only if it wins on the gate; reindex cost
   `[REINDEX_COST]` is part of the decision.

### Explicitly out of scope
Service layer, deployment, observability infrastructure, query rewriting,
anything in Phase 3.

### Definition of Done — Phase 2
- [ ] Recall@`[FINAL_K]` improves by ≥ `[P2_RECALL_DELTA]` over the Phase 1 baseline.
- [ ] Faithfulness and citation accuracy do not regress beyond `[P2_MAX_REGRESSION]`.
- [ ] Added p95 latency from reranking ≤ `[P2_LATENCY_BUDGET]`.
- [ ] Simulated Cohere outage (timeout + 5xx) degrades to dense-only with no query failures — drill evidence in the report.
- [ ] Flag flip verified in both directions without a restart.
- [ ] A/B report committed at `eval/reports/phase-2-ab.md`, including any regressions.
- [ ] `code-reviewer` Blocking findings: none outstanding.
- [ ] `[OWNER]` signs off.

> If the gate fails, report the real numbers and stop. Do not retune the
> threshold, the judge, or the dataset to manufacture a pass.

---

## Phase 3 — Production hardening & deployment

**Goal.** Operate the pipeline as a service on `[DEPLOY_TARGET]` with the
observability, resilience, and release safety to run unattended.

**Owner agent.** `platform-engineer`, with `eval-engineer` holding the gate.

### In scope
1. **Service layer** — `[API_STYLE]` (e.g. REST/FastAPI) exposing query,
   health, and readiness. Request validation, `[AUTH_METHOD]` authentication,
   rate limiting at `[RATE_LIMIT]`, structured error contracts.
2. **Observability** — structured JSON logs with a request ID threaded through
   every stage; metrics for latency (per stage), recall proxies, token and
   dollar cost, error and fallback rates; traces across ingest→retrieve→rerank→
   generate. Dashboards and alerts on `[ALERT_CONDITIONS]`. Never log secrets,
   and redact `[PII_FIELDS]` from logged payloads.
3. **Resilience** — timeouts and circuit breakers on every external call,
   bounded concurrency, backpressure, graceful shutdown. Answer caching with
   TTL `[CACHE_TTL]` where `[CACHE_SCOPE]` allows.
4. **Index lifecycle** — scheduled reindex at `[REINDEX_CADENCE]`, incremental
   where possible, blue/green index swap with rollback and a documented
   staleness SLO of `[STALENESS_SLO]`.
5. **CI/CD** — pipeline running lint, type check, unit and integration tests,
   secret scan, dependency audit, and a **subset eval gate** on every PR; full
   eval gate before deploy. Canary at `[CANARY_PERCENT]` with automatic
   rollback on `[ROLLBACK_TRIGGER]`.
6. **Load & chaos** — sustained load test at `[TARGET_RPS]` for
   `[LOAD_DURATION]`; chaos drills for vector-store unavailability, LLM
   provider 429/5xx, and Cohere outage. Results recorded.
7. **Runbook** — `docs/runbook.md`: architecture, config reference, deploy and
   rollback steps, alert-to-action table, common failures, on-call contacts.

### Definition of Done — Phase 3
- [ ] Service deployed to `[DEPLOY_TARGET]` and serving live traffic.
- [ ] Full eval gate passes post-deploy with no regression beyond `[P3_MAX_REGRESSION]` vs. Phase 2.
- [ ] p95 latency ≤ `[P3_LATENCY_TARGET]` and error rate ≤ `[P3_ERROR_BUDGET]` at `[TARGET_RPS]`.
- [ ] All three chaos drills pass; each degrades gracefully with alerts firing correctly.
- [ ] Canary + automatic rollback demonstrated end to end.
- [ ] Cost per 1k queries ≤ `[COST_CEILING]`, measured not estimated.
- [ ] Secret scan and dependency audit clean in CI.
- [ ] `docs/runbook.md` complete; a second engineer completes a dry-run deploy from it alone.
- [ ] `code-reviewer` Blocking findings: none outstanding.
- [ ] `[OWNER]` signs off.

---

## Filling this in

Replace every bracket with a decided value — not a guess and not a range.
Where a number is genuinely unknown until measured (latency and cost targets
especially), write the target you will hold the system to, and let the eval
gate tell you whether it was realistic. Record any target you later revise,
with the reason, in the corresponding `plans/phase-N-plan.md`.

Search for remaining placeholders before planning:

```
grep -n '\[[A-Z_]*\]' docs/rag-pipeline-phase-prompts.md
```
