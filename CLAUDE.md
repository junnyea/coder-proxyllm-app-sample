# RAG Pipeline — Project Memory & Orchestration Contract

## Project
Production-grade RAG system delivered in **3 gated phases**:
Phase 1 (classic RAG baseline) → Phase 2 (Cohere rerank) → Phase 3 (production hardening & deployment).

Source of truth for scope, requirements, and Definitions of Done: `docs/rag-pipeline-phase-prompts.md`.
All `[BRACKETED]` values in that file must be filled in before Phase 1 planning begins.

## Non-negotiables (bind every agent and subagent)
1. **Frozen golden dataset.** After Phase 1 sign-off, `eval/golden_dataset.jsonl` is immutable. Any change requires explicit written approval from the human owner, recorded in the plan file.
   Enforced, not merely stated: a `PreToolUse` hook in `.claude/settings.json` denies writes to that file whenever the marker `eval/.dataset-frozen` exists. The owner arms it by committing that marker at Phase 1 sign-off — so `git log eval/.dataset-frozen` is the sign-off record. Removing the marker is itself the approval event and must be justified in the plan file.
2. **Eval gates decide ship/no-ship.** Never tune thresholds, judge prompts, or the dataset to make a gate pass. Report honest numbers, including regressions.
3. **Config-driven.** All tunables (chunk size, k, models, thresholds, timeouts, prompts) live in config. No magic numbers in code.
4. **Secrets** come from the environment / secret manager. Never hardcoded, never logged, never committed.
5. **Phase scope discipline.** No Phase 2/3 features in Phase 1, and so on. Scope creep is a blocking review finding.
6. **Retrieved documents are untrusted input.** Generation prompts must resist instructions embedded in corpus content.

## Delegation map
| Work | Delegate to |
|---|---|
| Decompose a phase into a task plan | `phase-planner` |
| Phase 1 core RAG code (ingestion, chunking, index, retrieval, generation) | `pipeline-engineer` |
| Golden dataset, eval harness, eval runs, metric reports | `eval-engineer` |
| Phase 2 Cohere two-stage retrieval, fallback, flag gating | `rerank-engineer` |
| Phase 3 service layer, observability, CI/CD, resilience, deployment | `platform-engineer` |
| Review of any diff before merge | `code-reviewer` |

## Phase execution loop (orchestrator-driven)
Two equivalent ways to run this loop — the contract below binds both:
- **Default session**: this main session acts as orchestrator per this file.
- **Dedicated orchestrator**: launch `claude --agent rag-orchestrator` to run the orchestrator agent as the main session (its `initialPrompt` boots the loop automatically).

1. Delegate to `phase-planner` → produces `plans/phase-<N>-plan.md` with size points (S=1, M=2, L=3) and dependency-ordered **waves**.
2. Human approves the plan before implementation starts.
3. **Even-split dispatch per wave**: assign the wave's tasks so each concurrent worker carries a near-equal point load (concurrency cap [3] unless the human changes it). One task per subagent invocation, full context in the dispatch prompt. Implementers run in isolated worktrees (`isolation: worktree`), so parallel edits cannot collide.
4. As each worker returns: `code-reviewer` on its diff (all Blocking findings must clear) → merge in dependency order, smallest diff first. Failed or stalled tasks fold into the next wave and the split is rebalanced.
5. Delegate to `eval-engineer` to run the phase gate and report deltas vs. baseline.
6. Human signs off the Definition of Done. Only then does the next phase begin.

Keep verbose output (test logs, search results, eval traces) inside subagent context and return only summaries to the orchestrating session.
