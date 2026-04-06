# Benchmark: Pure Opus vs Opus+Codex

We ran the same tasks twice — once with Pure Opus (Claude does everything), once with opus-codex (Opus plans, Codex executes). Same codebase, same prompts, fresh sessions each time.

## Results (v1.5.2)

| | 80 LOC | | 400 LOC | | 1060 LOC | |
|---|---|---|---|---|---|---|
| | Pure Opus | opus-codex | Pure Opus | opus-codex | Pure Opus | opus-codex |
| **Cost** | $0.33 | $0.61 | $0.68 | $0.79 | $0.86 | $0.79 |
| **Cache read** | 261k | 482k | 383k | 544k | 430k | 455k |
| **Output tokens** | 3.1k | 4.5k | 5.5k | 9.8k | 13.6k | 6.8k |
| **Ratio** | — | 1.85x | — | 1.16x | — | **0.92x** |

## The crossover

opus-codex becomes cheaper than Pure Opus at ~800 LOC. Below that, the fixed overhead (planning, handoff, review) costs more than the code generation it saves.

| Task size | Recommendation | Why |
|-----------|---------------|-----|
| < 500 LOC | Pure Opus is cheaper | Planning overhead > execution savings |
| 500–800 LOC | Either approach | Roughly break-even |
| > 800 LOC | opus-codex saves ~9%+ | Output tokens drop ~50%, amortizing overhead |

## Where opus-codex wins

Output tokens tell the story. At 1060 LOC:
- Pure Opus: 13.6k output tokens (Opus writes all the code)
- opus-codex: 6.8k output tokens (Opus writes the plan, Codex writes the code)

Opus output tokens cost $75/M. Codex execution is free (or uses your OpenAI API key). The bigger the task, the more code generation shifts to Codex, and the more you save.

The sweet spot: mechanical, repetitive tasks where the plan is short but the output is large. E.g., "add error handling to all 30 API endpoints" or "migrate 50 files from class components to hooks."

## Where Pure Opus wins

Small tasks have structural overhead that can't be eliminated:
- Opus must read the codebase (same as Pure Opus)
- Opus must write the plan (extra step Pure Opus skips)
- Codex must re-read context from the plan (serialization cost)
- Opus must review the diff after Codex finishes

For an 80 LOC task, this overhead costs more than just letting Opus write the code directly.

## The fundamental tradeoff

Pure Opus thinks and acts in one step — it already has the context from reading the codebase.

opus-codex adds a serialization step: Opus must externalize its understanding into a plan document for Codex. This has inherent cost, but it pays off when the execution (code generation) is large enough to offset it.

## Methodology

- All runs on the same commit of main
- Fresh Claude Code sessions (no prior context)
- Identical prompts for both approaches
- Cost = (input tokens x $15/M) + (output tokens x $75/M) + (cache read x $1.875/M)
- Tasks: dry-run CLI feature (80 LOC), HTML report generator (400 LOC), jobs API with CRUD + tests (1060 LOC)
- Tested on tooltrust-scanner codebase (Python, ~5k LOC)
