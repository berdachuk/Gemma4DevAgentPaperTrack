## Abstract

Offline SWE agents under a hard tool budget often look busy and still fail: identical tool calls until the budget dies, empty diffs, or treating the model's own prose as a passing test.

We introduce a **budget-discipline layer** for closed-tool SWE agents: graph-first localization, typed corrective stops, and deterministic submission gates, with skill playbooks carrying procedure so the sticky system prompt stays short. Evidence is repro / pytest STATUS / `review.py` VERDICT — never assistant explanation. Operator iteration uses the same rule: one operator-level skills-package change per cycle from local ATIF aggregates (official prediction scoring returns no SessionTraces).

**Claim.** Under a fixed tool budget, discipline converts wasted exploration into editable patches. The measured win is a large cut in redundant calls and prompt tokens — not a resolve leap. After empty-submission mode clears, residual is `near_miss` / cousin paths. A locate sub-agent A/B was **dropped** (resolve flat, cheaper explore, no localization gain). Next licensed bet: a **tiny LoRA** via QLoRA on our own resolved / near-miss traces (smoke proven; score A/B pending).

On empty4 (skills-only, frozen `system.md` at 648 tok), a **candidate-path latch** cut median prompt tokens ~1.23M → ~555k and identical retries on `fastapi_15661` 33 → 1 (edits 0 → 5; resolve still 1/4). Latch turns budget collapse into an edit attempt; `fastapi_15661` still timed out at the local 300 s fence after five edits.

## 1. Introduction

The Gemma 4 Developer Agent competition runs a small offline coding agent against unfamiliar repositories. Empty patches score zero. Identical tool calls with the same arguments waste the budget. The harness exposes code graphs and embeddings as first-class navigation; recursive shell search alone is too slow to be the main plan.

This writeup targets **Best New Application**: declarative ADK skills, STATUS/NEXT routers, local ATIF classifiers. We never burn a prediction Submission for telemetry.

**What is new.** (1) Cache-stable prompt + skills that turn thrash into gated edits. (2) A measured **falsification list** for 12B. (3) A residual taxonomy that licenses LoRA from the loop's own traces — without claiming the score leap already happened.

**Portability.** Graph neighbors/subgraph (or equivalent), file R/W, status, deterministic test/STATUS, intercept submit diff before `submit_patch`.

## 2. Design constraints

- Closed tool surface: shell, file R/W, graph neighbors/subgraph, embedding search, status, `submit_patch`.
- Tight budgets: Host shared wall-clock is on the order of ~12 h across the offline task board; our local eval uses `max_time_minutes: 5.0` (~300 s) per task plus command timeouts of 300 s.
- Declarative ADK package: `agent.yaml`, prompts, Python skills (no custom `agent.py`).
- Cache-stable system prompt: hard ceiling of approximately 800 tokens (estimated as characters / 4), soft target 650. Procedure lives in skills (STATUS/NEXT). Current packaged prefix is ~600 tokens; the candidatelatch cycle below froze the prefix at 648.
- Competition model: one allowed alias (12B QAT checkpoint for serving). Adapters are allowed on that base; we do not rewrite the QAT file in place (§4.3).

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
| Cheap-loop | One operator-level skills-package change per cycle from aggregate traces + a named falsifier. |
| near_miss | Patch ships; hidden verifier summary has 1–2 failed, ≥1 passed, 0 collection errors. |

### Protocol (reproducibility)

| Field | Value |
|---|---|
| Primary falsifier | empty4 IDs: `fastapi_14964`, `fastapi_15588`, `fastapi_15589`, `fastapi_15661` |
| Compare dirs | before `results/shard-empty4-autorelated` → after `results/shard-empty4-candidatelatch` |
| Analyzer A/B | control `results/ab-ctrl-s1-20261112-101553` → treatment `results/ab-v2a-s1-20261112-110539`; decision `results/ab-s1-20261112-110539/ab_compare.json` |
| Runs / seeds | empty4: single-run observational; analyzer A/B: same 10-task shard, pre-registered drop rule |
| Prompt tokens | **Cumulative** ATIF `prompt_tokens` summed across model calls in the task (not per-request only) |
| Identical retries | Consecutive identical fingerprints only (§ Glossary) |
| Per-task wall clock | Local `max_time_minutes: 5.0` (~300 s) |
| Resolve | Official `resolved==true` |
| Correct-file | First edited product file appears among files changed in the reference/gold `patch` from the public task JSONL; multi-file partial credit not claimed here |
| Frozen prefix | candidatelatch cycle: `system.md` at 648 tokens (chars/4); current package ~600 |
| Larger shards | smoke10 `results/shard-smoke10-autorelated` (3/10); public20-regate `results/shard-live-public20-regate` (4/20) |
| LoRA smoke corpus | chat/tool SFT JSONL from own ATIF (`l2_chat`: 5 resolved + 4 near_miss + 4 locate pairs); sha-manifested; preference pairs reserved for later DPO |

### 3.1 Three-layer stack

```text
harness  ->  closed tool surface (+ graph / embedding tools)
loop     ->  budgeted trials + typed corrective stops
skills   ->  playbooks that encode task-family procedure
(+ adapter) -> optional LoRA on the same base (§4.3; WIP)
```

Figure 1. Budgeted graph-first SWE loop from harness tools through the trial loop and skill playbooks to evidence gates and `submit_patch`. An ATIF cheap-loop sits outside and applies one operator-level skills-package change per cycle.

Skill path: localize with graph/embed after `show` → reproduce → edit on a latched candidate → related tests / STATUS → `review.py` VERDICT → `submit_patch` only if non-empty.

**Design rule (12B, declarative ADK).** The only prompt/skill mechanism that reliably changed behavior was a **hard refusal with a concrete named NEXT tool** (and an exit path). Advice-only wording, temperature changes, and thinking-mode toggles did not (§4.2). Slogan: prompts teach manners; skills teach procedure; an adapter is a bet on **where** the model looks.

### 3.2 Mechanism map (package-level intervention)

The candidatelatch cycle is one **skills package**, not a factorial ablation:

| Change | Hypothesis | Primary metric |
|---|---|---|
| Candidate-path latch (`nav.py show`) | Stop losing the found target | first_edit latency; wrong-file edits |
| NEED_EDIT / FAIL NEXT → `edit_file '<candidate>'` | Edit instead of endless diagnose/grep | edits on failing tasks; empty_patch |
| Fingerprint stop | Stop identical tool loops | identical_tool_retries |
| Review-before-submit | Block empty / non-product / scope-drift diffs | empty_patch; VERDICT FIX FIRST rate |

We do **not** claim which sub-mechanism dominated the token/retry drop. Later same-shard A/B and lever reverts supply negative controls (§4.2); they are not a factorial decomposition of this package.

### 3.3 Evidence gates and `review.py` VERDICT

Gates are deterministic skill scripts (agent must run `review.py` before submit), not an LLM judge.

**Product `.py`:** changed path ends with `.py`, is not a test file, and is not grader-reset config (`conftest.py`, `pytest.ini`, `pyproject.toml`, `setup.cfg`, `tox.ini`, …).

**Related tests:** when the skill latch requires it, `tests.py related <product_path>` ranks tests by path/symbol overlap with the latched product file; review may auto-scout/auto-run the top hit. Absence of a current related run keeps FIX FIRST.

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

READY can still be semantically wrong (`wrong_edit` / `near_miss`); the gate targets ship discipline. On candidatelatch ATIF text, every empty4 task shows ≥1 `VERDICT: FIX FIRST` and ≥1 `VERDICT: READY` string match (string counts, not a formal precision study). **Over-block rate** (READY wrongly denied on a gold-valid diff) was not measured — a known risk (§6).

### 3.4 Cheap-loop from local traces

Raise score from `results/<run>/` ATIF, not from Kaggle SessionTraces. Aggregate first; open one or two failing logs; one operator-level skills-package change per cycle. The same traces later become the LoRA corpus (§4.3) — the loop feeds both skill edits and weight bets.

### 3.5 Worked cycle: candidate-path latch package

**Setup.** empty4 before/after as in the protocol. Prior `fastapi_15661` mostly grepped (41 grepish), never edited, 33 identical retries, ~1.37M prompt tokens.

**Change (one operator-level skills package).** `nav.py show` stores `candidate_path`; `diagnose.py` NEED_EDIT / FAIL NEXT names `edit_file '<candidate>'`. `system.md` stayed at 648 tokens for this cycle.

**Lifecycle (`fastapi_15661`, after).** issue → one `get_code_neighbors` (graph index 3; graph-before-edit) → `nav.py show` → NEED_EDIT → `edit_file` (first edit ~33 s; 5 edits) → tests/review pressure → **wall-clock timeout at local 5 min** (not empty submit). Identical retries 33 → 1; grepish 41 → 9. First edited file was `scripts/add_latest_release_date.py` vs gold `scripts/prepare_release.py` (cousin path — localization miss).

**empty4 per-task (single-run):**

| Task | Retries before→after | Edits before→after | Prompt tok before→after | Outcome after |
|---|---|---|---|---|
| `fastapi_14964` | 0→0 | 3→2 | 256k→112k | wrong_edit |
| `fastapi_15588` | 0→2 | 3→0 | 1.08M→997k | empty_patch |
| `fastapi_15589` | 3→0 | 16→2 | 1.46M→240k | resolved |
| `fastapi_15661` | **33→1** | **0→5** | 1.37M→870k | timeout |

Shard aggregates: resolve **1/4 flat**; graph_use 4/4 with **1 graph call before first edit** on each edited task; avg prompt ~1.04M → ~555k; **median** ~1.23M → ~555k; avg identical retries 9.0 → 0.75. The 33→1 figure is a **representative-task** case study.

**Manual localization audit (after):**

| Task | Graph-before-edit | Correct-file vs gold | Note |
|---|---|---|---|
| `fastapi_14964` | yes (1 graph call) | yes (`fastapi/responses.py`) | still `wrong_edit` (semantics) |
| `fastapi_15589` | yes (1 graph call) | yes (`fastapi/dependencies/utils.py`) | resolved |
| `fastapi_15661` | yes (1 graph call) | no (cousin script) | timeout after five edits |
| `fastapi_15588` | graph yes; **no edit** | gold was `fastapi/sse.py` (shown) | see below |

For `fastapi_15588`, localization succeeded: `nav.py show fastapi/sse.py` and repeated `STATUS: NEED_EDIT` / NEXT `edit_file 'fastapi/sse.py'`. The agent did **not** take that edit transition; it kept re-running diagnose/grep/repro until the 300 s fence, leaving empty_patch. So the residual is obedience to NEED_EDIT→edit, not missing graph.

Graph-before-edit is 3/3 on edited tasks; correct-file is **2/3**. Graph-before-edit establishes **ordering**, not necessity: the agent might have localized correctly without the graph call. We treat graph-first as a supported workflow prior, not a proven causal localization mechanism.

## 4. Discipline ladder across local shards

We do **not** claim a finished three-way ablation or a same-task-set cost–quality frontier. What we reproduce is a **discipline ladder** under a frozen system prompt — buying cost and behavioral reliability before accuracy. Rows use different shards; do not read them as paired A/B on one ID set.

| Stage (local) | Tasks | Resolve | graph_use | empty_patch | Cost / loop signal |
|---|---|---:|---:|---:|---|
| Pre-discipline smoke10 | 10 | 3/10 | 1/10 | — | grepish storms; graph ignored |
| empty4 graphfix | 4 | 0–1/4 | ≥0.5 | **0/4** | nonempty median patch; graph_use gate ≥0.5 |
| candidatelatch (§3.5) | 4 | 1/4 | 4/4 | **1/4** | 15661 identical 33→1; median prompt ~555k |
| fail_empty | 4 | 1/4 | 4/4 | **0/4** | empty-submission mode off on this falsifier |
| smoke10 autorelated | 10 | 3/10 | 10/10 | **0/10** | avg prompt ~354k on related empty4 lineage |

Resolve stayed roughly flat on larger falsifiers (smoke10 **3/10**, public20-regate **4/20**). Process wins: real graph calls, fewer identical retries, lower prompt tokens; empty_patch **0/4** / **0/10** on fail_empty and smoke10 autorelated. We iterate from local ATIF, not prediction Submissions (no SessionTraces).

### 4.1 Residual after empty-submission mode is off

Once `empty_patch` is **0/4** (fail_empty) and **0/10** (smoke10) on these local shards, delivery/discipline is largely solved. On a merged champion-era taxonomy (n=28 public rolls; gate artifact `results/gate-p0a-merged`):

| bucket | n |
|---|---:|
| resolved | 4 |
| **near_miss** | **4** |
| wrong_edit | 3 |
| verify_fail | 2 |
| infra | 2 |
| timeout | 13 |

`near_miss` is the informative residual: almost clears hidden (1–2 fails). Example `rich_3953`: wants `rich/cells.py`, patches adjacent `rich/_emoji_replace.py` — same cousin pattern as empty4 `fastapi_15661`. A cousin-clone READY block catches one write pattern only. Discipline (`empty_patch`+loop) was **0**; localization mass licensed A/B (§4.2). Timeouts 13/28 remain a 5 min confound.

### 4.2 Falsified levers (post-discipline)

Each lever below was implemented, measured, and reverted (or never kept). Cost control: identical failure signature twice → stop.

1. **Remove `read_file`** (force nav-only reads): starvation deadlock; edits collapsed. Reverted.
2. **Thinking mode ON**: served chat template never emits a thought channel; tokens wasted. Reverted.
3. **Temperature ≠ 0**: resolved sets identical across temperatures — attractors are policy properties, not sampling noise. Reverted.
4. **Advice-only stops** ("consider reviewing"): ignored; only hard refusals with named NEXT change behavior. Reverted.
5. **Related-first detour** (run related tests before edit): gate fired 0× in practice; first edit delayed; a solved task lost. Reverted.
6. **Budgeted read-only locate sub-agent** (one `agent_tool`, root one-call cap): same-shard A/B vs champion control. Verdict **DROP** — resolved sets identical (**1/10**, only `rich_3454`); decision-class (resolved + near_miss) **−1**; one-call cap violated once (`fastapi_11355` with 2 calls). Mean tool calls **41.2 → 30.0** and mean duration **289 → 246 s**: delegation replaced the explore storm (contradicting naive turn-cost fear) but did **not** purchase localization quality. Wiring reverted; silent re-enable blocked without new evidence.

Takeaway: **more structure is not free accuracy**. Soft advice is exhausted; open class = localization — weights or better locate signal.

### 4.3 Next licensed bet: LoRA from own traces (WIP)

Design + smoke readiness — **not** a resolve leap. Skills cleared empty/loop modes; near_miss ≈ resolved (4/4 on n=28); structure A/B bought no locate quality; ≤800-tok advice does not move 12B. Training data is a cheap-loop byproduct.

**Twin vs race car.** Serve competition **QAT** 12B alias. Train a **twin** (same instruct family, float weights) via **QLoRA** (4-bit NF4) on RTX 4090 24 GB, seq 4096, LoRA ranks ~32. Bolt a thin adapter onto the QAT race car — do **not** rewrite the QAT file.

**Corpus (own ATIF only):** resolved positives; near-miss as negative context (not “correct” targets); locate pairs naming verifier failing test files only; preference pairs reserved for later DPO. Smoke `l2_chat`: **13 rows** (5+4+4), sha-manifested — proves path/VRAM, not a leaderboard bump (trainer warns <~50).

**Keep rule.** Ship adapter only after fixed-shard with/without A/B improves resolve **and** discipline gates still pass. Until then: reproducible WIP, not a scored claim.

## 5. Related work and positioning

Closest neighbors: SWE-agent (Yang et al., 2024), OpenHands / OpenDevin (Wang et al., 2024), Agentless-style localize→repair→validate (Xia et al., 2024); SWE-bench (Jimenez et al., 2024); LoRA / QLoRA (Hu et al., 2021; Dettmers et al., 2023).

Our niche: **budget-aware operational discipline on a closed harness** — hard STATUS/VERDICT stops with named NEXT, skills carrying procedure for a cache-stable prefix, an operator cheap-loop that refuses prediction Submissions as telemetry, a measured **falsification list**, and a weight lane fed by the same traces once structure levers plateau.

## 6. Limitations

- Small *n* and single-run comparisons on empty4; report median/min–max.
- Candidatelatch is a **package**, not a factorial ablation; §4.2 falsifies whole levers, not sub-mechanisms inside the latch package.
- Correct-file localization is only 2/3 on edited empty4 tasks; graph-before-edit ≠ graph-attributable necessity.
- Latch does not guarantee finish inside 300 s (`fastapi_15661` timeout after edits).
- NEED_EDIT obedience can still fail (`fastapi_15588`).
- `review.py` over-block rate unmeasured; gate can miss semantic wrong edits / near_misses.
- Hidden tests / publicScore can diverge from local shards; ATIF proxies imperfect without SessionTraces.
- Analyzer A/B is one shard / one wiring; negative result may not transfer to other models or budgets.
- LoRA §4.3 is smoke/path only: 13-row corpus; no publicScore claim; adapter keep-gate not yet fired.
- Edit quality (`near_miss` / cousin localization) remains open after empty-submission mode is cleared on these shards.

## 7. Conclusion

Gates beat a longer system prompt for thrash and empty patches. Discipline buys token/retry cuts at flat resolve, then falsifies structural locate fixes. The next copyable piece is a **tiny adapter on the loop's own near-misses** — same keep/drop rigor as skills, not another advice paragraph in `system.md`.

## Acknowledgments

Competition harness and Paper Track rules are those published by the Host. Writeup materials are Apache-2.0–friendly for winner-license reproducibility.
