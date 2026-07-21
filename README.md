# Claude Code Agent Kit — Production RAG (3 Phases)

Project-scoped subagents that let Claude Code delegate the RAG build: planning, implementation, evaluation gating, review, and production hardening.

## What's inside
```
.claude/
└── agents/
    ├── phase-planner.md       # decomposes each phase into an owned task plan
    ├── pipeline-engineer.md   # Phase 1: ingestion, chunking, index, retrieval, generation
    ├── eval-engineer.md       # golden dataset + eval harness; the phase gatekeeper
    ├── rerank-engineer.md     # Phase 2: Cohere two-stage retrieval + fallback
    ├── platform-engineer.md   # Phase 3: service, observability, CI/CD, deploy
    └── code-reviewer.md       # read-only review gate before every merge
CLAUDE.md                      # orchestration contract: non-negotiables, delegation map, phase loop
plans/                         # phase-planner writes phase-N-plan.md here
```

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
- **isolation: worktree**: give an implementer its own temporary git worktree for riskier tasks.
- Agents are plain Markdown + YAML frontmatter — edit freely. Reference: https://code.claude.com/docs/en/sub-agents
