## Abstract

Offline SWE agents under a hard tool budget often look busy and still fail. The usual failure modes we hit were: repeating the same tool call until the budget dies, submitting an empty diff, or treating the model's own explanation as if it were a passing test.

We built around a three-layer stack: the closed harness tools, a budgeted trial loop with typed stops, and skill playbooks. Evidence comes from deterministic gates (repro, pytest STATUS, review VERDICT), not from an LLM judge. On the Gemma 4 Developer Agent harness that means graph-first localization, review before `submit_patch`, and a failure taxonomy that keeps infra bugs out of prompt-tuning cycles.

We apply the same rule to our own iteration. A cheap-loop reads aggregate local ATIF traces (Kaggle prediction scoring does not return SessionTraces) and is allowed one cache-friendly change per cycle. On the public empty4 falsifier, a skill-side candidate-path latch cut identical-tool retries from 33 to 1 and average prompt tokens from about 1.04M to about 555k. The system prompt was not edited. Resolve on that shard stayed flat; the claim is discipline and cost, not a resolve leap.

## 1. Introduction

The Gemma 4 Developer Agent competition runs a small offline coding agent against unfamiliar repositories. The agent must land test-passing patches under a shared wall-clock budget and tight per-task tool limits. Empty patches score zero. Identical tool calls with the same arguments waste the budget fast. The harness exposes code graphs and embeddings as first-class navigation; recursive shell search alone is too slow to be the main plan.

This writeup targets the **Best New Application** lane: a method you can copy under a similar harness contract — declarative ADK skills, STATUS/NEXT routers, and local ATIF classifiers. Because official prediction scoring does not return SessionTraces, we never burn a prediction Submission for telemetry. Measurement stays on local shards and ATIF traces.

Paper Track only; the prediction `submission.zip` is a separate deliverable. Public Project Link (no login, no paywall): https://github.com/berdachuk/Gemma4DevAgentPaperTrack

## 2. Design constraints

The harness fixes several design choices:

- Closed tool surface: shell, file R/W, graph neighbors/subgraph, embedding search, status, `submit_patch`.
- Tight budgets: ~12 h shared across the task board; local per-task fences on the order of minutes and tens of tool calls.
- Declarative ADK package: `agent.yaml`, prompts, and Python skills (no custom `agent.py`).
- Cache-stable system prompt: hard ceiling 800 tokens by chars/4, soft target 650. Procedure belongs in skills (STATUS/NEXT). Do not rewrite the prompt head or middle for date, run id, or other ephemeral fields.

## 3. Methods

### Glossary

| Term | Meaning |
|---|---|
| ATIF | Local agent-trace format under `results/<run>/` (tool calls, timings, outcomes). |
| empty4 | Four-task public falsifier shard used as a cheap gate before larger evals. |
| graph_use | Fraction of tasks with a real graph tool call (`get_code_neighbors` / subgraph), not a text mention. |
| Evidence gate | Pass/fail from external checks (repro, pytest STATUS, review VERDICT). Assistant prose never counts. |
| Cheap-loop | One measured change per cycle from aggregate traces, paired with a falsifier. |

### 3.1 Three-layer stack

```text
harness  ->  closed tool surface (+ graph / embedding tools)
loop     ->  budgeted trials + typed corrective stops
skills   ->  playbooks that encode task-family procedure
```

Figure 1. Budgeted graph-first SWE loop from harness tools through the trial loop and skill playbooks to evidence gates and `submit_patch`. An ATIF cheap-loop sits outside and picks one lever per cycle. Diagram: Media gallery (`fig1-budgeted-graph-first-swe-loop.png`).

In practice the skills push the agent through:

1. Localize with graph or embed after `show` (grepish thrash is the failure mode we keep seeing).
2. Reproduce the failing signal.
3. Edit once on a latched candidate path.
4. Run related tests; read STATUS.
5. Review before submit (VERDICT).
6. `submit_patch` only if the diff is non-empty.

Stops that fire in our runs:

- Same tool and args twice → fingerprint stop.
- `NO_MATCH` after `show` → re-localize; do not invent a path.
- Empty patch → refuse submit.
- Review fail → edit again.

### 3.2 Failure taxonomy and shard gate

Classify the failure before touching prompts or stacking Analyzer/LoRA:

`empty_patch`, `no_locate`, `wrong_edit`, `verify_fail`, `infra`, `timeouts`, plus scratch or protected-file leaks.

The shard gate answers a simple question: harness/infra bug, or agent bug? Analyzer and LoRA wait until the gate says proceed. Skipping that step is how you "fix" the model while the eval wiring is still wrong.

### 3.3 AI-assisted cheap-loop from local traces

We raise prediction score from local `results/<run>/` ATIF traces and logs. Official Kaggle SessionTraces are not part of that loop.

1. Aggregate first: `analyze_eval_traces --list`, `classify_eval_results`, then a deeper `analyze_eval_traces`.
2. Open one or two failing logs, not the whole shard dump.
3. Stage-1 parquet alone is patch-only; there is no trajectory there.
4. Do not assume a prediction submit returns traces. It does not, in our reading of the contract.

| Signal | Prefer next fix |
|---|---|
| low `graph_use` / high grepish | nav/graph after `show` / `NO_MATCH` |
| high `first_edit_s` | edit after one strong `show` |
| `identical_tool_retries` | skill fingerprint loop / shell fingerprint |
| high edits + low resolve | review before submit |
| high prompt tokens | cut re-reads; analyzer only after gate proceed |
| high `empty_rate` | ship a patch earlier; then shard why |
| taxonomy `no_locate` + proceed | analyzer AgentTool plan |

One lever per cycle. Operator writeup of the cycle: Verdict, a 1-3 task counterfactual, the single next change, why it is cheaper or more cache friendly, the falsifier, and what is out of scope.

Things that repeatedly wasted cycles: pasting every trace into chat; growing `system.md` instead of moving text into skills; rewriting the mid-prompt for a date; stacking LoRA, prompt edits, and analyzer in the same cycle.

### 3.4 Worked cycle: candidate-path latch

Setup: empty4 falsifier at `results/shard-empty4-candidatelatch`. On `fastapi_15661` the prior run mostly grepped, never edited, and died on budget.

Change (one lever), skills only. `system.md` stayed at 648 tokens.

- `nav.py show` stores `candidate_path`
- `diagnose.py` NEED_EDIT / FAIL NEXT names `edit_file '<candidate>'`

| Metric | Before to after |
|---|---|
| Resolve (empty4) | 1/4 (flat) |
| `graph_use` | 4/4 = 1.0 |
| empty_patch (gate) | ≤1 (G2-G4 go) |
| `fastapi_15661` edits | 0 to 5 |
| grepish calls | 41 to 9 |
| identical tool retries | 33 to 1 |
| prompt tokens (15661) | ~1.37M to ~870k |
| first_edit | null to 32s |
| avg empty4 prompt | ~1.04M to ~555k |

Resolve did not jump on this shard. Behavior did: edits appeared, retries collapsed, tokens dropped. Residuals still in the mix: timeout-after-edit, `empty_patch`, `wrong_edit`. No prediction Submission was used to collect these traces.

## 4. Experiments

Section 3.4 is the primary measured falsifier. The same metrics apply on every shard: resolve rate, `empty_patch` rate, tool-call histograms, `graph_use_rate`, first-edit elapsed, `identical_tool_retries`, average prompt tokens, failure-taxonomy mix. Falsifiers stay small (`--task-id`, empty4, overlay). Full public20 is not the default every cycle.

We do **not** claim a finished three-way public ablation (no-graph / graph-without-skills / full). What we can reproduce week to week is a **discipline ladder** under a frozen system prompt (skills-only levers):

| Stage (local) | Resolve | graph_use | empty_patch | Cost / loop signal |
|---|---|---|---|---|
| Pre-discipline smoke10 | 3/10 | 1/10 | — | grepish storms; graph tools ignored |
| empty4 graphfix | 0–1/4 | ≥0.5 | 0 | G2–G4 go; nonempty patches |
| candidatelatch (§3.4) | 1/4 | 4/4 | ≤1 | identical 33→1; avg prompt ~555k |
| fail_empty / autorelated | 1/4 · smoke10 3/10 | 4/4 · **10/10** | 0 | avg prompt ~354k; empty floor cleared |

Resolve on these public falsifiers stayed roughly flat (smoke10 **3/10**, public20 freeze **4/20**). The win is process: real graph calls, fewer identical retries, lower prompt tokens, empty-patch gate go. Official prediction publicScore on early burns sat around **0.05–0.06**; we treat that as an external ceiling and keep iterating from local ATIF, not from submit-for-traces.

### 4.1 Residual after the empty floor

Once `empty_patch` is near zero, the freeze taxonomy is dominated by **`wrong_edit`** (7 ids across smoke10 + public20-regate), then infra and timeouts. That is the intended next cheap-loop target (skills A/B on a `wrong_edit` shard). Analyzer AgentTool and LoRA stay deferred until the shard gate says proceed — stacking them earlier burned budget without clearing the floor.

## 5. Related work and positioning

Closest neighbors are budgeted tool-using coding agents: SWE-agent (Yang et al., 2024), OpenHands / OpenDevin (Wang et al., 2024), and Agentless-style pipelines that cut free-form tool thrash (Xia et al., 2024). SWE-bench (Jimenez et al., 2024) set the repository-level patch yardstick many harnesses now share. Graph and embedding retrieval show up in several stacks next to ordinary search.

Where we spend pages is narrower. STATUS/VERDICT gates are **hard stops**, not an LLM-as-judge submit step. Skills carry procedure so the sticky system prompt stays short and cache-stable — the alternative we keep rejecting is growing the prompt or jumping to RL/PEFT before the empty-patch floor is clear. The operator cheap-loop reads local ATIF aggregates, changes one lever, and refuses prediction Submissions as a telemetry source.

## 6. Limitations

- Hidden tests may not look like our public shards; local resolve (3/10, 4/20) can diverge from leaderboard publicScore.
- Official prediction runs return no SessionTraces; ATIF proxies are necessary but imperfect.
- The discipline ladder is sequential one-lever cycles, not a factorial ablation on the hidden board.
- Edit quality (`wrong_edit`) remains the open problem after empty-patch discipline.

## 7. Conclusion

Empty patches and tool thrash are cheaper to prevent with gates and skill stops than with a longer system prompt. Keep procedure in skills so the sticky prefix stays small. When the host does not return SessionTraces, raise score from local ATIF aggregates and change one thing per cycle.

On this harness, that is the loop we run for Gemma 4: graph-first localization where the tools allow it, evidence before submit, and a measured cheap-loop outside the agent. After the empty floor, the residual is wrong edits — still one lever at a time.

## Acknowledgments

Competition harness and Paper Track rules are those published by the Host on the competition Website. Writeup materials for this entry are Apache-2.0–friendly for winner-license reproducibility (Project Link in §1).
