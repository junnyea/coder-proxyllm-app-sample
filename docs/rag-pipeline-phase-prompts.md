# RAG Pipeline — Phase Prompts

**Status:** READY FOR PLANNING — all values decided.
Defaults below were chosen to be defensible starting points, not immutable law.
Revise any of them deliberately, and record the change and its reason in the
corresponding `plans/phase-N-plan.md`. Never revise a *target* to make a
failing gate pass.

This file is the source of truth for scope, requirements, and Definitions of
Done. The orchestration contract (who does what, in what order) lives in
`CLAUDE.md` and does not repeat scope. Where the two ever conflict, this file
wins on *what* and `CLAUDE.md` wins on *how*.

---

## 0. Shared context (binds all three phases)

| Decision | Value |
|---|---|
| Use case / question domain | Internal engineering documentation Q&A — "how does our system X work", "what's the runbook for Y", "which service owns Z" |
| Primary users | Internal engineers and on-call responders (~50 seats) |
| Corpus source & location | Internal docs repo (Markdown), exported Confluence pages, and service READMEs — mirrored to `s3://eng-docs-corpus/` |
| Corpus size (docs / tokens) | ~4,000 documents / ~12M tokens |
| Document formats | Markdown, HTML, plain text, PDF |
| Update cadence | Nightly batch |
| Embedding model | `voyage-3-large` (1024-dim, 32k context) |
| Generation model | `claude-sonnet-5` |
| Vector store | PostgreSQL 16 + `pgvector` (HNSW index, cosine distance) |
| Language(s) | English only |
| Deployment target | AWS ECS Fargate (`us-east-1`), behind an ALB |
| Data sensitivity / compliance regime | Internal-confidential. No customer PII in the corpus; SOC 2 controls apply |
| Cost ceiling per 1k queries | $6.00 (all providers, retrieval + generation combined) |
| Human owner (approves gates) | `hoongjun@gmail.com` |

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
- Secrets from environment / AWS Secrets Manager only. Never hardcoded, logged, or committed.
- Retrieved documents are **untrusted input**; generation prompts must resist
  instructions embedded in corpus content.
- Every tunable (chunk size, overlap, `k`, model IDs, thresholds, timeouts,
  prompt templates) is config-declared and overridable per environment.
- No phase may implement a later phase's features. Scope creep is a Blocking
  review finding.

---

## Phase 1 — Classic RAG baseline

**Goal.** A working, measured, end-to-end RAG pipeline that answers internal
engineering-docs questions grounded in the mirrored corpus, plus the frozen
evaluation apparatus that every later phase is judged against.

**Owner agents.** `pipeline-engineer` (pipeline), `eval-engineer` (dataset +
harness). Both review through `code-reviewer`.

### In scope
1. **Ingestion** — load Markdown, HTML, text, and PDF from `s3://eng-docs-corpus/`;
   normalize to text; strip nav chrome, footers, and boilerplate; preserve
   per-document metadata: `source_id`, `title`, `url_or_path`, `ingested_at`,
   plus `owning_team`, `doc_type`, `last_modified`, `heading_path`.
   Deterministic and idempotent: re-running over an unchanged corpus produces
   an identical index.
2. **Chunking** — structural, heading-aware splitting that falls back to
   recursive-character within oversized sections. Target size **512 tokens**,
   overlap **64 tokens**. Every chunk carries its parent document metadata, its
   `heading_path`, and a stable content-hash chunk ID.
3. **Indexing** — embed with `voyage-3-large`; persist to pgvector with an HNSW
   index (`m=16`, `ef_construction=64`). Batched at 128 chunks per request,
   resumable from the last committed batch, with backoff on 429s.
   Index build is a single repeatable command.
4. **Retrieval** — dense top-**10** cosine similarity search with metadata
   filtering on `owning_team` and `doc_type`. Single-stage only;
   **no reranking in Phase 1**.
5. **Generation** — assemble retrieved context into a grounded prompt for
   `claude-sonnet-5`. Must cite `source_id` per claim, must refuse when context
   is insufficient rather than answering from parametric memory, and must
   delimit chunk text as data rather than instructions.
6. **Golden dataset** — `eval/golden_dataset.jsonl`, **150 questions**, mixed
   as: 45% factual lookup, 20% multi-hop, 15% comparison, 10% procedural
   ("how do I…"), 10% unanswerable. Each record: `question`,
   `ground_truth_answer`, `relevant_chunk_ids`, `question_type`, `difficulty`.
7. **Eval harness** — reproducible run producing retrieval metrics (recall@k,
   MRR, nDCG@10), generation metrics (faithfulness, answer relevance, citation
   accuracy), and latency + cost per query. Judge model **`claude-opus-4-8`**,
   judge prompts versioned in config. Output: a dated report under
   `eval/reports/`.

### Explicitly out of scope
Reranking, hybrid/sparse retrieval, query rewriting, multi-turn conversation,
HTTP service layer, authentication, autoscaling, CI/CD.

### Definition of Done — Phase 1
- [ ] `make ingest && make index && make query` run clean from a bare checkout.
- [ ] Recall@10 ≥ **0.75** on the golden dataset.
- [ ] Faithfulness ≥ **0.90**; citation accuracy ≥ **0.85**.
- [ ] Unanswerable questions are refused ≥ **90%** of the time.
- [ ] p95 end-to-end latency ≤ **5.0 s**.
- [ ] Baseline report committed at `eval/reports/phase-1-baseline.md`.
- [ ] Zero secrets in the diff; all tunables in `config/`.
- [ ] `code-reviewer` Blocking findings: none outstanding.
- [ ] **`hoongjun@gmail.com` signs off — at which point `eval/golden_dataset.jsonl` is FROZEN.**

> After sign-off the golden dataset is immutable. Any later change needs written
> approval from the owner, recorded in the relevant `plans/phase-N-plan.md`.

---

## Phase 2 — Cohere rerank (two-stage retrieval)

**Goal.** Raise retrieval precision by over-retrieving then reranking, proven by
an honest A/B against the frozen Phase 1 baseline.

**Owner agents.** `rerank-engineer` (implementation), `eval-engineer` (A/B).

### In scope
1. **Over-retrieval** — widen stage-1 dense retrieval to **50** candidates
   (from 10).
2. **Rerank** — **`rerank-v3.5`** scores candidates; keep top **5** above
   relevance threshold **0.30**. If nothing clears the threshold, the pipeline
   reports insufficient context rather than padding with weak matches.
3. **Fallback & resilience** — timeout **2,000 ms**, **2** retries with
   exponential backoff (250 ms base, jittered). On failure, degrade gracefully
   to Phase 1 dense-only top-10 and emit a structured warning with the request
   ID. A Cohere outage must never fail a query.
4. **Flag gating** — `retrieval.rerank_enabled` in config toggles reranking at
   runtime, defaulting to **`true`** in all environments. Both paths stay
   exercised by tests.
5. **A/B evaluation** — same frozen dataset, same judge, rerank on vs. off.
   Report deltas on every Phase 1 metric plus added latency and Cohere cost.
6. **Optional experiment** — swap `voyage-3-large` for **`embed-v4.0`** at
   stage 1 and measure. Ships only if it wins the gate; the ~**$40 + 3 h**
   full-reindex cost is part of the decision.

### Explicitly out of scope
Service layer, deployment, observability infrastructure, query rewriting,
anything in Phase 3.

### Definition of Done — Phase 2
- [ ] Recall@5 improves by ≥ **+0.08 absolute** over the Phase 1 baseline.
- [ ] Faithfulness and citation accuracy do not regress by more than **0.02 absolute**.
- [ ] Added p95 latency from reranking ≤ **400 ms**.
- [ ] Simulated Cohere outage (timeout + 5xx) degrades to dense-only with no query failures — drill evidence in the report.
- [ ] Flag flip verified in both directions without a restart.
- [ ] A/B report committed at `eval/reports/phase-2-ab.md`, including any regressions.
- [ ] `code-reviewer` Blocking findings: none outstanding.
- [ ] Owner signs off.

> If the gate fails, report the real numbers and stop. Do not retune the
> threshold, the judge, or the dataset to manufacture a pass.

---

## Phase 3 — Production hardening & deployment

**Goal.** Operate the pipeline as a service on ECS Fargate with the
observability, resilience, and release safety to run unattended.

**Owner agent.** `platform-engineer`, with `eval-engineer` holding the gate.

### In scope
1. **Service layer** — REST API (FastAPI) exposing `POST /query`,
   `GET /healthz`, `GET /readyz`. Request validation, authentication via
   **OIDC JWT from the internal IdP**, rate limiting at **20 req/min per user,
   200 req/min per service token**, structured error contracts.
2. **Observability** — structured JSON logs with a request ID threaded through
   every stage; metrics for per-stage latency, recall proxies, token and dollar
   cost, error and fallback rates; OpenTelemetry traces across
   ingest→retrieve→rerank→generate, exported to CloudWatch + Grafana.
   Alerts on: p95 latency > 6 s for 5 min, error rate > 2% for 5 min, rerank
   fallback rate > 10% for 10 min, nightly reindex failure, daily spend > $50.
   Never log secrets; redact `user_email`, `jwt`, and raw query text from
   logged payloads at `WARN` and below.
3. **Resilience** — timeouts and circuit breakers on every external call
   (Voyage, Anthropic, Cohere, Postgres), concurrency bounded at **32 in-flight
   queries**, backpressure via 429 with `Retry-After`, graceful shutdown
   draining in-flight work. Answer caching with TTL **1 h**, keyed by
   normalized query **plus the caller's `owning_team` filter set** so cached
   answers never cross an access boundary.
4. **Index lifecycle** — scheduled reindex **nightly at 02:00 UTC**,
   incremental by `last_modified` with a weekly full rebuild; blue/green index
   swap via table alias with one-command rollback; documented staleness SLO of
   **≤ 24 h**.
5. **CI/CD** — GitHub Actions running lint, type check, unit and integration
   tests, secret scan, dependency audit, and a **30-question subset eval gate**
   on every PR; full 150-question gate before deploy. Canary at **10%** of
   traffic for 30 min, with automatic rollback on error rate > 2% or p95
   latency > 8 s sustained 5 min.
6. **Load & chaos** — sustained load test at **25 RPS** for **30 min**; chaos
   drills for pgvector unavailability, Anthropic 429/5xx, and Cohere outage.
   Results recorded in the phase report.
7. **Runbook** — `docs/runbook.md`: architecture, config reference, deploy and
   rollback steps, alert-to-action table, common failures, on-call contacts.

### Definition of Done — Phase 3
- [ ] Service deployed to ECS Fargate and serving live traffic.
- [ ] Full eval gate passes post-deploy with no metric regressing more than **0.02 absolute** vs. Phase 2.
- [ ] p95 latency ≤ **6.0 s** and error rate ≤ **1%** at 25 RPS.
- [ ] All three chaos drills pass; each degrades gracefully with alerts firing correctly.
- [ ] Canary + automatic rollback demonstrated end to end.
- [ ] Cost per 1k queries ≤ **$6.00**, measured not estimated.
- [ ] Secret scan and dependency audit clean in CI.
- [ ] `docs/runbook.md` complete; a second engineer completes a dry-run deploy from it alone.
- [ ] `code-reviewer` Blocking findings: none outstanding.
- [ ] Owner signs off.

---

## Revising these values

Scope and configuration values (chunk size, `k`, cadences, models) may be
revised whenever evidence justifies it — record the change and its reason in
the active plan file.

**Targets and thresholds are different.** Once Phase 1 is signed off, the
metric targets and the golden dataset are the yardstick. Moving a target
because a gate failed invalidates every comparison that came before it. Report
the real number, and let the owner decide whether to accept the miss, fix the
system, or consciously re-baseline.

Verify no placeholders remain before planning:

```
grep -n '\[[A-Z_]*\]' docs/rag-pipeline-phase-prompts.md
```
