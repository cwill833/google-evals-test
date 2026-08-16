# Eval Structure & Conventions — datasets, tiers, and Google's premade metrics

**Audience:** the build agent (encode these conventions in the glue/lint) and every future eval author.
**Ground truth:** the 17 built-in metrics below were listed live from the pinned toolchain
(`agents-cli eval metric list`, 2026-08-16, agents-cli 1.3.1). Metric names are written lowercase in
configs/flags (verified live: `--metrics final_response_quality` worked).

---

## 1. The one rule everything hangs on

**A dataset file's `<tier>-` filename prefix binds it to exactly one grading profile** (its
`eval_config.<tier>.yaml` — the metrics + thresholds applied at the `grade` step; `generate` itself is
metric-blind). A tier = that grading profile **plus a case-authoring contract** (what the cases must
contain for the profile's scores to mean anything — enforced by the lint). Cases never carry metrics; files do, via their prefix. The prefix→config
mapping lives in ONE place — `tools/eval_tiers.py` — imported by the CI loop, the Runner worker, and
the lint. Adding a tier = one config template + one mapping line. Nothing else changes.

```
agents/<agent>/tests/eval/
  datasets/
    fast-smoke.json            fast-tooling.json
    judged-quality.json        judged-instruction.json
    golden-answers.json
    grounded-lookups.json
    safety-probes.json
    multiturn-support-flow.json
  eval_config.fast.yaml        ← one config per tier IN USE by this agent
  eval_config.judged.yaml         (agents only carry the tiers they use)
  eval_config.golden.yaml
  ...
  response_quality.py          ← local custom-judge functions, if any
```

## 2. The tier taxonomy — mapped to Google's premade metrics

> **Live availability check (2026-08-16, build-spec L12):** the eval-service backend currently accepts
> only a SUBSET of the registry. ✅ Working: `final_response_quality`, `final_response_match`, local
> custom functions. ❌ Rejected (`Unsupported predefined metric`, both regions): `grounding`,
> `hallucination`, `safety`, `instruction_following`, `tool_use_quality`. Tiers marked *(dormant)*
> below are structurally ready but ship no config until the registry watch shows their metrics live.

| Tier prefix | Metric suite (premade unless noted) | Reference req'd? | Cost | CI posture |
|---|---|---|---|---|
| `fast-` | local `custom_function` metrics only (turn counts, tool-call assertions, deterministic checks) | no | free, seconds | **blocking**, every merge |
| `judged-` | `final_response_quality` (✅ live) + the scaffold's local `custom_response_quality` as a second opinion. *(`instruction_following`, `tool_use_quality` join when the backend supports them — L12)* | no | judge calls | **blocking** (owner decision) — generous thresholds, tighten after baseline month |
| `golden-` | `final_response_match` (✅ live — semantic match to golden reference) | **YES — every case, linted** | judge calls | blocking |
| `grounded-` *(dormant — L12)* | `grounding`, `hallucination` (response vs tool outputs — cases must exercise tools) | no | judge calls | blocking when live |
| `safety-` **(ACTIVE — owner decision 2026-08-16)** | **local custom red-team judge** (your rubric: refuse/deflect correctly on attack prompts — same `custom_function` pattern as `response_quality.py`); Google's premade `safety` metric *augments* it when the backend supports it (L12) | no | judge calls | blocking |
| `multiturn-` | `multi_turn_general_quality`, `multi_turn_task_success`, `multi_turn_tool_use_quality`, `multi_turn_trajectory_quality`, `multi_turn_text_quality` (pick per config; cases carry conversation via `agent_data`/`conversation_history`) | no | judge calls, priciest | start report-only; promote when stable |

Premade metrics not mapped: `general_quality`, `text_quality`, `final_response_reference_free`
(overlap with `judged-` picks — adopt into `eval_config.judged.yaml` if wanted), `gecko_text2image/video`
(media agents — future tier `media-` when one exists).

**Why `golden-` is its own tier and not part of `judged-`:** the reference requirement. We watched
Google's own SDK crash on mixed reference/no-reference datasets (build-spec L7) and the metric mean
silently poisons if references are sparse (review G6). Making "requires reference" a *tier property*
turns a subtle data bug into a lint error at PR time.

## 3. When to add a case vs. a new file vs. a new tier

1. **New case, existing suite** — same tier, fits the file's theme → append to `eval_cases`. IDs are
   `<topic>-<behavior>-<nn>`, kebab-case, stable forever (they're join keys in the GCS results — never
   reuse or rename a shipped ID; retire by deletion).
2. **New file, same tier** — same metrics, new theme, or a file passing ~50 cases (reviewability
   split): `fast-tool-edge-cases.json`.
3. **New tier** — the case needs a *different metric suite* → new prefix + config template + one
   mapping line. Rare by design.

## 4. Keeping N agents consistent (scalability)

- **Templates:** repo-root `eval-templates/eval_config.<tier>.yaml` are the canonical suites. A new
  agent copies the tiers it needs. The lint flags an agent config that drifts from its template unless
  it carries an explicit `# custom-suite: <reason>` header — divergence is allowed, but visible.
- **Lint (`tools/eval_lint.py`, runs in PR checks, no credentials):**
  - every dataset parses as `EvaluationDataset` (schema in build-spec E6/fixtures)
  - every dataset's prefix has a config in that agent + mapping entry
  - `golden-*`: every case has `reference` (the G6 rule)
  - case IDs unique within the agent; no ID reuse vs `main`
  - judged-family datasets report case count (judge-cost visibility in the PR)
- **Ownership:** CODEOWNERS — `datasets/fast-*` eng-only; `judged-/golden-/safety-` rubric-bearing
  tiers require product + eng review (arch §15.5 resolved by path convention).
- **Metric registry watch:** a committed `eval-metrics.snapshot` (output of `eval metric list`); CI on
  any toolchain version bump diffs live output against it and surfaces new premade metrics as a PR
  comment. Google shipped 17 metrics as of today; they keep adding — this is how we notice.
- **Gate:** per-tier `thresholds:` blocks (verified tolerated in configs) with `min_mean`/`min_case`;
  gate also enforces graded-case-count == dataset-case-count (build-spec L2 — silent-drop protection).

## 5. Future lanes that slot in WITHOUT redesign

- **Baseline-relative gating:** persist per-agent `baseline.json` (last main-run scores) next to the
  datasets; `eval compare` diffs runs today; gate v2 can flag regression-vs-baseline instead of only
  absolute thresholds. Additive to the gate script.
- **Synthesized datasets / user simulation:** `eval dataset synthesize` (experimental) generates
  multi-turn candidate cases → curate → commit as `multiturn-*` files. Same convention, no new
  machinery. Agent Engine's managed Evaluation Layer + User Simulator is the eventual managed version.
- **Cloud-side evals:** `eval submit`/`results` (shelved — backend 404 today, build-spec L7) will slot
  in as an alternative *execution* path when it matures; datasets/configs/tiers are unchanged by it.
- **Prompt optimization:** `eval optimize` (GEPA) consumes the same datasets — the eval corpus doubles
  as training signal for instruction tuning. No structural cost now.
- **Conformance/replay regression:** `adk conformance` record/replay is a separate lane
  (`tests/conformance/`) — deterministic replay, complements rather than joins the tier system.

## 6. CI ordering & cost control

Run tiers cheapest-first, fail fast: `fast` → `judged` → `golden`/`grounded`/`safety` → `multiturn`.
Per-tier `--concurrency` on generate bounds parallel judge/model pressure. The judged tiers run only
on merge to main and on-demand — never per-commit (arch §8 rule 4). Every run's per-tier scores land
in the same GCS results layout regardless of tier, so the UI and trends need no tier-specific code.
