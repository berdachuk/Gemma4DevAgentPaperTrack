## Abstract

Offline SWE agents under a hard tool budget often look busy and still fail: identical tool calls until the budget dies, empty diffs, or treating the model's own prose as a passing test.

We introduce a **budget-discipline layer** for closed-tool SWE agents: graph-first localization, typed corrective stops, and deterministic submission gates, with skill playbooks carrying procedure so the sticky system prompt stays short. Evidence is `repro` / pytest `STATUS` / `review.py` `VERDICT` — never assistant explanation. Operator iteration uses the same rule: one cache-friendly change per cycle from local ATIF aggregates (official prediction scoring returns no SessionTraces).

**Claim.** Under a fixed tool budget, disciplined localization and evidence-gated submission convert wasted exploration into editable patches. The immediate measured effect is a large cut in redundant tool calls and prompt-token cost — the precondition for later edit-quality work — not a resolve leap on small public shards.

On a four-task public empty4 shard (single-run observational comparison, skills-only, frozen `system.md` at 648 tokens), a candidate-path latch package moved average prompt tokens from ~1.04M to ~555k (median ~1.23M → ~555k; min–max after: 112k–997k). On the representative failing task `fastapi_15661`, identical tool retries fell from 33 to 1 and edits appeared (0 → 5). Shard resolve stayed 1/4.

## 1. Introduction

The Gemma 4 Developer Agent competition runs a small offline coding agent against unfamiliar repositories under a shared wall-clock budget and tight per-task tool limits. Empty patches score zero. Identical tool calls with the same arguments waste the budget. The harness exposes code graphs and embeddings as first-class navigation; recursive shell search alone is too slow to be the main plan.

This writeup targets **Best New Application**: a copyable method under a similar harness contract — declarative ADK skills, STATUS/NEXT routers, and local ATIF classifiers. We never burn a prediction Submission for telemetry.

Paper Track only; prediction `submission.zip` is separate. Public Project Link (no login, no paywall): https://github.com/berdachuk/Gemma4DevAgentPaperTrack

## 2. Design constraints

- Closed tool surface: shell, file R/W, graph neighbors/subgraph, embedding search, status, `submit_patch`.
- Tight budgets: ~12 h shared across the board; local fences on the order of minutes and tens of tool calls.
- Declarative ADK package: `agent.yaml`, prompts, Python skills (no custom `agent.py`).
- Cache-stable system prompt: hard ceiling 800 tokens by chars/4, soft 650. Procedure lives in skills (STATUS/NEXT). Do not rewrite the prompt head/middle for ephemeral fields.

## 3. Methods

### Glossary

| Term | Meaning |
|---|---|
| ATIF | Local agent-trace format under `results/<run>/` (tool calls, timings, outcomes). |
| empty4 | Fixed four-task public shard used as a **cheap falsifier**: if a one-lever change does not move the predicted loop metric (retries / empty_patch / graph-before-edit) in the expected direction, reject the change before larger evals. |
| identical retry | Consecutive (or near-consecutive) tool calls with the same name and normalized args; counted from ATIF. |
| graph_use | Fraction of tasks with ≥1 real graph tool call. **Compliance**, not usefulness — see §3.1. |
| Evidence gate | Deterministic external check; assistant prose never counts. |
| Cheap-loop | One measured change per cycle from aggregate traces + a named falsifier. |

### Protocol (reproducibility)

| Field | Value |
|---|---|
| Primary falsifier | empty4 task IDs: `fastapi_14964`, `fastapi_15588`, `fastapi_15589`, `fastapi_15661` |
| Compare dirs | before `results/shard-empty4-autorelated` → after `results/shard-empty4-candidatelatch` |
| Runs / seeds | Single-run observational (no multi-seed); treat as case study + shard aggregates |
| Prompt tokens | Sum of ATIF prompt tokens per task; report avg, median, and min–max |
| Resolve | Official task `resolved==true` |
| Frozen prefix | `system.md` unchanged at 648 tokens (skills-only lever) |
| Larger shards | smoke10 `results/shard-smoke10-autorelated` (3/10); public20-regate `results/shard-live-public20-regate` (4/20) |

### 3.1 Three-layer stack

```text
harness  ->  closed tool surface (+ graph / embedding tools)
loop     ->  budgeted trials + typed corrective stops
skills   ->  playbooks that encode task-family procedure
```

Figure 1. Budgeted graph-first SWE loop from harness tools through the trial loop and skill playbooks to evidence gates and `submit_patch`. An ATIF cheap-loop sits outside and picks one lever per cycle.

Skill path in practice: localize with graph/embed after `show` → reproduce → edit once on a latched candidate → related tests / STATUS → `review.py` VERDICT → `submit_patch` only if non-empty.

`graph_use` alone is weak: a skill can force a call. Prefer **graph-before-edit** (graph tool index &lt; first edit index) and whether an edit appears after localization. On candidatelatch, 3/3 edited tasks had graph-before-edit; `fastapi_15588` called graph but never edited (empty_patch residual).

### 3.2 Mechanism map (conceptual levers)

The candidatelatch cycle was one **skills package**, not a factorial ablation. Conceptually:

| Change | Hypothesis | Primary metric |
|---|---|---|
| Candidate-path latch (`nav.py show`) | Stop losing the found target | first_edit latency; wrong-file edits |
| NEED_EDIT / FAIL NEXT → `edit_file '<candidate>'` | Edit instead of endless diagnose/grep | edits on failing tasks; empty_patch |
| Fingerprint stop | Stop identical tool loops | identical_tool_retries |
| Review-before-submit | Block empty / non-product / scope-drift diffs | empty_patch; VERDICT FIX FIRST rate |

We do **not** claim which sub-mechanism dominated the token/retry drop; negative controls are future work (§6).

### 3.3 Evidence gates and `review.py` VERDICT

Gates are deterministic skill scripts, not an LLM judge:

1. **Repro / pytest STATUS** — failing signal before edit; STATUS after related tests.
2. **`review.py` VERDICT** — inspects the exact diff `submit_patch` would capture. Emits `VERDICT: READY` or `VERDICT: FIX FIRST` from rules including: empty diff; no product `.py`; docs-only / dirty docs noise; edits to grader-reset test/config files; compile failures; leftover debug prints; product edits outside the latched path (scope drift); missing verify / related-test evidence. READY can still be wrong on semantics (`wrong_edit`); the gate targets ship discipline, not patch correctness.
3. **Empty-patch refuse** — skills forbid `submit_patch` on EMPTY_DIFF / FIX FIRST.

### 3.4 Cheap-loop from local traces

Raise score from `results/<run>/` ATIF, not from Kaggle SessionTraces (unavailable on prediction scoring). Aggregate first (`analyze_eval_traces`, `classify_eval_results`); open one or two failing logs; one lever per cycle; write Verdict / counterfactual / falsifier / out-of-scope.

Wasted cycles we keep rejecting: pasting every trace into chat; growing `system.md`; mid-prompt date rewrites; stacking LoRA + prompt + analyzer in one cycle.

### 3.5 Worked cycle: candidate-path latch

**Setup.** empty4 before/after as in the protocol table. Prior `fastapi_15661` mostly grepped (41 grepish), never edited, 33 identical retries, ~1.37M prompt tokens, died without a usable edit loop.

**Change (one skills package).** `nav.py show` stores `candidate_path`; `diagnose.py` NEED_EDIT / FAIL NEXT names `edit_file '<candidate>'`. `system.md` stayed at 648 tokens.

**Lifecycle (`fastapi_15661`, after).** issue load → `get_code_neighbors` (graph index 3) → `nav.py show` latches path → NEED_EDIT → `edit_file` (first edit ~33s; 5 edits total) → tests / review pressure → wall-clock **timeout** (not empty submit). Identical retries 33 → 1; grepish 41 → 9.

**empty4 per-task (single-run):**

| Task | Retries before→after | Edits before→after | Prompt tok before→after | Outcome after |
|---|---|---|---|---|
| `fastapi_14964` | 0→0 | 3→2 | 256k→112k | wrong_edit |
| `fastapi_15588` | 0→2 | 3→0 | 1.08M→997k | empty_patch |
| `fastapi_15589` | 3→0 | 16→2 | 1.46M→240k | resolved |
| `fastapi_15661` | **33→1** | **0→5** | 1.37M→870k | timeout |

Shard aggregates: resolve **1/4 flat**; graph_use 4/4; avg prompt ~1.04M → ~555k; **median** ~1.23M → ~555k; avg identical retries 9.0 → 0.75. The 33→1 figure is a **representative-task** case study, not a four-task mean of that magnitude.

## 4. Experiments: budget-efficiency frontier

We do **not** claim a finished three-way ablation (no-graph / graph-without-skills / full). What we reproduce week to week is a **discipline ladder** under a frozen system prompt — buying cost and behavioral reliability before accuracy:

| Stage (local) | Resolve | graph_use | empty_patch | Cost / loop signal |
|---|---|---|---|---|
| Pre-discipline smoke10 | 3/10 | 1/10 | — | grepish storms; graph ignored |
| empty4 graphfix | 0–1/4 | ≥0.5 | 0 | G2–G4 go; nonempty patches |
| candidatelatch (§3.5) | 1/4 | 4/4 | ≤1 | 15661 identical 33→1; median prompt ~555k |
| fail_empty / autorelated | 1/4 · smoke10 3/10 | 4/4 · 10/10 | 0 | avg prompt ~354k; empty floor cleared |

Resolve on these public falsifiers stayed roughly flat (smoke10 **3/10**, public20 freeze **4/20**). The frontier move is process: real graph calls, fewer identical retries, lower prompt tokens, empty-patch gate go. Official prediction publicScore on early burns (~0.05–0.06) is an **external ceiling signal only** (few submits; not a multi-run estimate) — we do not use submits for traces.

### 4.1 Residual after the empty floor

Once `empty_patch` is near zero, unresolved mass is **`wrong_edit`** (5 on smoke10; 4 on public20-regate), then infra/timeouts. That is the next cheap-loop target. Analyzer AgentTool and LoRA stay deferred until the shard gate says proceed.

## 5. Related work and positioning

Closest neighbors: SWE-agent (Yang et al., 2024), OpenHands / OpenDevin (Wang et al., 2024), Agentless-style localize→repair→validate (Xia et al., 2024); SWE-bench (Jimenez et al., 2024) as the patch yardstick. Graph/embedding retrieval appears in several stacks beside ordinary search.

Our niche is narrower: **budget-aware operational discipline on a closed harness** — hard STATUS/VERDICT stops (not LLM-as-judge submit), skills carrying procedure so the sticky prefix stays cache-stable, and an operator cheap-loop that refuses prediction Submissions as telemetry.

## 6. Limitations

- Small *n* (4 / 10 / 20) and single-run comparisons; one heavy task can move averages — report median/min–max.
- No factorial ablation of latch vs fingerprint vs review (§3.2); candidatelatch is one package.
- `graph_use` can be compliance; prefer graph-before-edit and edit appearance.
- `review.py` can block bad ships and still miss semantic wrong edits; it can also over-block.
- Hidden tests may diverge from public shards; ATIF proxies imperfect without SessionTraces.
- Edit quality (`wrong_edit`) remains open after the empty floor.

## 7. Conclusion

Empty patches and tool thrash are cheaper to prevent with gates and skill stops than with a longer system prompt. Under a fixed budget, the measured win of this cycle is elimination of redundant exploration and a large prompt-token cut at flat resolve — clearing the floor so edit-quality levers can be tested one at a time.

## Acknowledgments

Competition harness and Paper Track rules are those published by the Host. Writeup materials are Apache-2.0–friendly for winner-license reproducibility (Project Link in §1).
