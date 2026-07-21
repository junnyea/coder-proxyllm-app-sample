# Claude Code Agent Kit — Production RAG (3 Phases)

Project-scoped subagents that let Claude Code delegate the RAG build: planning, implementation, evaluation gating, review, and production hardening.

## What's inside
```
.claude/
└── agents/
    ├── rag-orchestrator.md    # main-session agent: even-split parallel dispatch, merge, gates
    ├── phase-planner.md       # decomposes each phase into sized tasks + balanced waves
    ├── pipeline-engineer.md   # Phase 1: ingestion, chunking, index, retrieval, generation
    ├── eval-engineer.md       # golden dataset + eval harness; the phase gatekeeper
    ├── rerank-engineer.md     # Phase 2: Cohere two-stage retrieval + fallback
    ├── platform-engineer.md   # Phase 3: service, observability, CI/CD, deploy
    └── code-reviewer.md       # read-only review gate before every merge
CLAUDE.md                      # orchestration contract: non-negotiables, delegation map, phase loop
plans/                         # phase-planner writes phase-N-plan.md here
```

## Run with the orchestrator (even task splitting)
```
claude --agent rag-orchestrator
```
This runs the orchestrator as the main session agent: its `initialPrompt` reads the contract, reports phase status, and proposes the next **wave** — tasks sized in points (S=1, M=2, L=3) and split near-evenly across up to [3] parallel workers. Implementers carry `isolation: worktree`, so concurrent edits happen in separate git worktrees; each worker hands back a branch + diff summary, the reviewer clears it, and the orchestrator merges. You approve plans and phase gates; the orchestrator never implements.

Prefer a plain session? The same loop is written into `CLAUDE.md`, so a normal `claude` session orchestrates identically — the dedicated agent just boots it for you.

## Install
1. Unzip at your repository root so `.claude/agents/` and `CLAUDE.md` sit at the top level (merge `CLAUDE.md` content if you already have one).
2. Place your phase prompts file at `docs/rag-pipeline-phase-prompts.md` and fill in every `[BRACKETED]` value — the planner refuses to plan until they're decided.
3. If `.claude/agents/` did not exist before your current Claude Code session started, restart the session once so the directory is picked up. After that, file edits hot-reload.
4. Commit the kit to version control so the whole team shares the same agents.

## Use
Start a phase:
```
Start Phase 1 — use the phase-planner subagent to produce the plan.
```
Then let the orchestration loop in `CLAUDE.md` drive: implementer → code-reviewer → tests → eval-engineer gate → your sign-off. Claude also delegates automatically based on each agent's `description`; you can always invoke one explicitly ("Have the eval-engineer run the Phase 2 comparison").

## Customization levers
- **model** per agent: add `model: haiku|sonnet|opus|fable` (or a full model ID) in the frontmatter. Default is `inherit` — every agent runs on your session's model. Common tweak: pin `code-reviewer` to a cheaper model, or the planner to a stronger one.
- **tools / disallowedTools**: tighten or widen access; reviewer is deliberately unable to Write/Edit.
- **isolation: worktree** (already set on the three implementers): each run gets a temporary git worktree, which is what makes parallel implementation safe. Remove the line if you prefer strictly sequential, in-place edits with no merge step.
- **Concurrency cap**: the `[3]` parallel-worker cap lives in `CLAUDE.md` and `rag-orchestrator.md` — raise it if your tasks are small and disjoint, lower it to reduce token burn.
- **maxTurns**: optionally bound any agent (e.g., `maxTurns: 50`) as a runaway-cost guard.
- Agents are plain Markdown + YAML frontmatter — edit freely. Reference: https://code.claude.com/docs/en/sub-agents
