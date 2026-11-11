## Abstract

Offline SWE agents under a hard tool budget often look busy and still fail: identical tool calls until the budget dies, empty diffs, or treating the model's own prose as a passing test.

We introduce a **budget-discipline layer** for closed-tool SWE agents: graph-first localization, typed corrective stops, and deterministic submission gates, with skill playbooks carrying procedure so the sticky system prompt stays short. Evidence is `repro` / pytest `STATUS` / `review.py` `VERDICT` — never assistant explanation. Operator iteration uses the same rule: one cache-friendly change per cycle from local ATIF aggregates (official prediction scoring returns no SessionTraces).

**Claim.** Under a fixed tool budget, disciplined localization and evidence-gated submission convert wasted exploration into editable patches. The immediate measured effect is a large cut in redundant tool calls and prompt-token cost — the precondition for later edit-quality work — not a resolve leap on small public shards.

On a four-task public shard (single-run observational comparison, skills-only, frozen `system.md` at 648 tokens), a **skills-only candidate-path latch package** reduced median prompt tokens from ~1.23M to ~555k (avg ~1.04M → ~555k; min–max after: 112k–997k) and eliminated repeated identical calls on the representative failing task `fastapi_15661` (33 → 1; edits 0 → 5), while shard resolve remained 1/4.

## 1. Introduction

The Gemma 4 Developer Agent competition runs a small offline coding agent against unfamiliar repositories. Empty patches score zero. Identical tool calls with the same arguments waste the budget. The harness exposes code graphs and embeddings as first-class navigation; recursive shell search alone is too slow to be the main plan.

This writeup targets **Best New Application**: a copyable method under a similar harness contract — declarative ADK skills, STATUS/NEXT routers, and local ATIF classifiers. We never burn a prediction Submission for telemetry.

**Portability.** Minimal harness requirements to reproduce the loop: graph neighbors/subgraph (or equivalent structured nav), file R/W, status, a deterministic test/STATUS signal, and the ability to intercept the would-be submit diff before `submit_patch`.

Paper Track only; prediction `submission.zip` is separate. Public Project Link (no login, no paywall): https://github.com/berdachuk/Gemma4DevAgentPaperTrack

## 2. Design constraints

- Closed tool surface: shell, file R/W, graph neighbors/subgraph, embedding search, status, `submit_patch`.
- Tight budgets: Host shared wall-clock is on the order of ~12 h across the offline task board; our local eval uses `max_time_minutes: 5.0` (~300 s) per task plus command timeouts of 300 s.
- Declarative ADK package: `agent.yaml`, prompts, Python skills (no custom `agent.py`).
- Cache-stable system prompt: hard ceiling 800 tokens by chars/4, soft 650. Procedure lives in skills (STATUS/NEXT).

## 3. Methods

### Glossary

| Term | Meaning |
|---|---|
| ATIF | Local agent-trace format under `results/<run>/` (tool calls, timings, outcomes). |
| empty4 | Fixed four-task public shard used as a **cheap falsifier**: if a skills-package change does not move the predicted loop metric (retries / empty_patch / graph-before-edit) in the expected direction, reject before larger evals. |
| identical retry | Count of times fingerprint(tool name + normalized args) equals the **immediately previous** tool call (consecutive only; ATIF). |
| grepish | Shell `run_command` lines matching `\b(grep\|rg\|find)\b`. |
| graph_use | Fraction of tasks with ≥1 real graph tool call. **Compliance**, not usefulness. |
| Evidence gate | Deterministic external check; assistant prose never counts. |
| Cheap-loop | One measured operator change per cycle from aggregate traces + a named falsifier. |

### Protocol (reproducibility)

| Field | Value |
|---|---|
| Primary falsifier | empty4 IDs: `fastapi_14964`, `fastapi_15588`, `fastapi_15589`, `fastapi_15661` |
| Compare dirs | before `results/shard-empty4-autorelated` → after `results/shard-empty4-candidatelatch` |
| Runs / seeds | Single-run observational (no multi-seed) |
| Prompt tokens | **Cumulative** ATIF `prompt_tokens` summed across model calls in the task (not per-request only) |
| Identical retries | Consecutive identical fingerprints only (§ Glossary) |
| Per-task wall clock | Local `max_time_minutes: 5.0` (~300 s) |
| Resolve | Official `resolved==true` |
| Frozen prefix | `system.md` unchanged at 648 tokens (skills-only package) |
| Larger shards | smoke10 `results/shard-smoke10-autorelated` (3/10); public20-regate `results/shard-live-public20-regate` (4/20) |

### 3.1 Three-layer stack

```text
harness  ->  closed tool surface (+ graph / embedding tools)
loop     ->  budgeted trials + typed corrective stops
skills   ->  playbooks that encode task-family procedure
```

Figure 1. Budgeted graph-first SWE loop from harness tools through the trial loop and skill playbooks to evidence gates and `submit_patch`. An ATIF cheap-loop sits outside and picks one operator change per cycle.

Skill path: localize with graph/embed after `show` → reproduce → edit on a latched candidate → related tests / STATUS → `review.py` VERDICT → `submit_patch` only if non-empty.

### 3.2 Mechanism map (package-level intervention)

The candidatelatch cycle is one **skills package**, not a factorial ablation:

| Change | Hypothesis | Primary metric |
|---|---|---|
| Candidate-path latch (`nav.py show`) | Stop losing the found target | first_edit latency; wrong-file edits |
| NEED_EDIT / FAIL NEXT → `edit_file '<candidate>'` | Edit instead of endless diagnose/grep | edits on failing tasks; empty_patch |
| Fingerprint stop | Stop identical tool loops | identical_tool_retries |
| Review-before-submit | Block empty / non-product / scope-drift diffs | empty_patch; VERDICT FIX FIRST rate |

We do **not** claim which sub-mechanism dominated the token/retry drop; negative controls are future work (§6).

### 3.3 Evidence gates and `review.py` VERDICT

Gates are deterministic skill scripts (agent must run `review.py` before submit), not an LLM judge.

**Product `.py`:** changed path ends with `.py`, is not a test file, and is not grader-reset config (`conftest.py`, `pytest.ini`, `pyproject.toml`, `setup.cfg`, `tox.ini`, …).

**Machine-checkable sketch:**

```text
READY iff ALL of:
  diff != empty
  >= 1 product .py changed
  no grader-reset test/config file in the ship set
  no compile failure on changed .py
  no debug prints (print/breakpoint/pdb) in added lines
  no unresolved scope drift outside latched candidate
  verify evidence present (post-edit diagnose OK)
  related-test evidence present when required by skill latch
else FIX FIRST
```

“Missing verify / related-test evidence” means the session latch still expects `diagnose` OK and/or a current `tests.py related` run. READY can still be semantically wrong (`wrong_edit`); the gate targets ship discipline. **Over-block rate** (READY wrongly denied on a gold-valid diff) was not measured on empty4 — a known risk (§6).

### 3.4 Cheap-loop from local traces

Raise score from `results/<run>/` ATIF, not from Kaggle SessionTraces. Aggregate first; open one or two failing logs; one operator change per cycle.

### 3.5 Worked cycle: candidate-path latch package

**Setup.** empty4 before/after as in the protocol. Prior `fastapi_15661` mostly grepped (41 grepish), never edited, 33 identical retries, ~1.37M prompt tokens.

**Change (skills package).** `nav.py show` stores `candidate_path`; `diagnose.py` NEED_EDIT / FAIL NEXT names `edit_file '<candidate>'`. `system.md` stayed at 648 tokens.

**Lifecycle (`fastapi_15661`, after).** issue → `get_code_neighbors` (graph index 3) → `nav.py show` → NEED_EDIT → `edit_file` (first edit ~33 s; 5 edits) → tests/review pressure → **wall-clock timeout at local 5 min** (not empty submit). Identical retries 33 → 1; grepish 41 → 9. First edited file was `scripts/add_latest_release_date.py` vs gold `scripts/prepare_release.py` (cousin path — localization miss).

**empty4 per-task (single-run):**

| Task | Retries before→after | Edits before→after | Prompt tok before→after | Outcome after |
|---|---|---|---|---|
| `fastapi_14964` | 0→0 | 3→2 | 256k→112k | wrong_edit |
| `fastapi_15588` | 0→2 | 3→0 | 1.08M→997k | empty_patch |
| `fastapi_15589` | 3→0 | 16→2 | 1.46M→240k | resolved |
| `fastapi_15661` | **33→1** | **0→5** | 1.37M→870k | timeout |

Shard aggregates: resolve **1/4 flat**; graph_use 4/4; avg prompt ~1.04M → ~555k; **median** ~1.23M → ~555k; avg identical retries 9.0 → 0.75. The 33→1 figure is a **representative-task** case study.

**Manual localization audit (edited tasks after):**

| Task | Graph-before-edit | Correct-file vs gold | Note |
|---|---|---|---|
| `fastapi_14964` | yes | yes (`fastapi/responses.py`) | still `wrong_edit` (semantics) |
| `fastapi_15589` | yes | yes (`fastapi/dependencies/utils.py`) | resolved |
| `fastapi_15661` | yes | no (cousin script) | timeout after edits |
| `fastapi_15588` | graph yes; **no edit** | — | graph compliance without edit routing — empty_patch residual |

So graph-before-edit is 3/3 on edited tasks; correct-file is **2/3**. Graph-first is partially supported as localization help, not merely as a forced call.

## 4. Discipline ladder across local shards

We do **not** claim a finished three-way ablation or a same-task-set cost–quality frontier. What we reproduce is a **discipline ladder** under a frozen system prompt — buying cost and behavioral reliability before accuracy. Rows use different shards; do not read them as paired A/B on one ID set.

| Stage (local) | Tasks | Resolve | graph_use | empty_patch | Cost / loop signal |
|---|---|---:|---:|---:|---|
| Pre-discipline smoke10 | 10 | 3/10 | 1/10 | — | grepish storms; graph ignored |
| empty4 graphfix | 4 | 0–1/4 | ≥0.5 | **0/4** | submit-go G2–G4 (empty≤1, median patch>0, graph_use≥0.5) |
| candidatelatch (§3.5) | 4 | 1/4 | 4/4 | **1/4** | 15661 identical 33→1; median prompt ~555k |
| fail_empty | 4 | 1/4 | 4/4 | **0/4** | empty floor cleared on this falsifier |
| smoke10 autorelated | 10 | 3/10 | 10/10 | **0/10** | avg prompt ~354k on related empty4 lineage |

Resolve stayed roughly flat on larger public falsifiers (smoke10 **3/10**, public20-regate **4/20**). Process wins: real graph calls, fewer identical retries, lower prompt tokens, empty-patch discipline on the small falsifiers.

| Signal | Value | Interpretation |
|---|---:|---|
| Official publicScore (early burns) | ~0.05–0.06 | External ceiling; few submissions; **not** used for iteration |
| Local smoke10 resolve | 3/10 | Primary local quality signal |
| Local public20-regate resolve | 4/20 | Secondary local quality signal |

Local resolve (15–30% on these shards) and early publicScore need not match: different mixes, hidden tests, and early-burn packaging. We keep iterating from ATIF, not from submit-for-traces.

### 4.1 Residual after the empty floor

Once `empty_patch` is **0/4** (fail_empty) and **0/10** (smoke10), unresolved mass is **`wrong_edit`** (5 on smoke10; 4 on public20-regate), then infra/timeouts. Next cheap-loop target. Analyzer/LoRA stay deferred until the shard gate says proceed.

## 5. Related work and positioning

Closest neighbors: SWE-agent (Yang et al., 2024), OpenHands / OpenDevin (Wang et al., 2024), Agentless-style localize→repair→validate (Xia et al., 2024); SWE-bench (Jimenez et al., 2024).

Our niche: **budget-aware operational discipline on a closed harness** — hard STATUS/VERDICT stops, skills carrying procedure for a cache-stable prefix, and an operator cheap-loop that refuses prediction Submissions as telemetry.

## 6. Limitations

- Small *n* and single-run comparisons; report median/min–max.
- Candidatelatch is a **package**, not a factorial ablation; no negative control yet.
- Correct-file localization is only 2/3 on the edited empty4 tasks; graph-before-edit ≠ graph-attributable necessity.
- `review.py` over-block rate unmeasured; gate can miss semantic wrong edits.
- Hidden tests / publicScore can diverge from local shards; ATIF proxies imperfect without SessionTraces.
- Edit quality (`wrong_edit`) remains open after the empty floor.

## 7. Conclusion

Empty patches and tool thrash are cheaper to prevent with gates and skill stops than with a longer system prompt. Under a fixed budget, the measured win of this cycle is elimination of redundant exploration and a large prompt-token cut at flat resolve — clearing the empty floor so edit-quality levers can be tested one package at a time.

## Acknowledgments

Competition harness and Paper Track rules are those published by the Host. Writeup materials are Apache-2.0–friendly for winner-license reproducibility (Project Link in §1).
