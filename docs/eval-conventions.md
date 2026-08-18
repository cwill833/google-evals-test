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
contain for the profile's scores to mean anything — enforced by the lint).

Rev-4 clarifications: a grading profile **may run multiple compatible metrics**. Golden references are
**human-approved semantic targets, not exact-string expectations** — exact matching is reserved for
strict deterministic contracts (JSON fields, calculations, identifiers, mandated disclosures), which
belong in `fast-` checks. Reference-based metrics run only when every case carries the required
reference; reference-free LLM judges need no golden answer. The prefix resolves through a **centrally
versioned mapping** — prefix → grading profile ID → metric/version/input/execution requirements
(`tools/eval_tiers.py`) — so tier meanings cannot drift across agents. Custom `custom_function`
metrics are **trusted code**: they execute on the laptop and in CI (which already run repo code), but
never in the credential-bearing Runner — Apex runs report them as `not_run` (run = `partial`) until a
sandbox exists. Cases never carry metrics; files do, via their prefix. The prefix→config
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
| `judged-` | `final_response_quality` (✅ live) + the scaffold's local `custom_response_quality` as a second opinion. *(`instruction_following`, `tool_use_quality` join when the backend supports them — L12)* | no | judge calls | **participates day one; promoted to blocking on baseline stability evidence** (rev 4 — conservative thresholds, fixed canaries, rollback policy) |
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

## 2b. Where should an eval go? (decision guide for agent builders)

Ask these in order — the first "yes" decides the file prefix:

1. **Can a plain function decide it?** Tool called with the right args, JSON schema valid, required
   field present, forbidden action absent, turn count, calculation → **`fast-`**. No judge, no cost,
   no noise. **When in doubt, prefer this** — a deterministic check is always better than a judge if
   one can express the requirement.
2. **Is it adversarial — testing that the agent refuses, deflects, or protects?** Jailbreaks, policy
   bait, privacy probes → **`safety-`**. Pass = refusing well, graded by our refusal rubric
   (Google's managed safety metric augments the profile when their backend enables it).
3. **Is there a single correct answer a human can approve?** → **`golden-`**. Write the reference as
   a semantic target (meaning, not exact wording); every case in the file must carry one — linted.
4. **Otherwise it's a quality judgment with no fixed answer** → **`judged-`**. Reference-free LLM
   grading plus case-specific rubrics.
5. Later, when live-validated: claims must be supported by tool evidence → `grounded-`;
   whole-conversation behavior → `multiturn-`.

**Who grades each profile** (the judging model — ownership/review routing is §4's CODEOWNERS):

| Profile | Grader |
|---|---|
| `fast-` | no judge — deterministic functions |
| `judged-` | Google's managed judge (`final_response_quality`) + our custom rubric as second opinion |
| `golden-` | Google's managed judge (`final_response_match`, semantic vs the approved reference) |
| `safety-` | **our** custom refusal rubric (Google's managed safety metric augments when enabled) |
| `grounded-` / `multiturn-` | Google's managed judges — dormant until the backend enables them |

**The rule of thumb:** Google's judges wherever they exist and are live; our judges where theirs
don't exist yet or where the criteria are ours to define; no judge at all when a function can decide.
As Google enables more managed metrics (the registry watch detects this), our custom judges shrink
toward the domain-specific remainder — that is the paved-road bet at the judge layer.

**Worked examples:**
- "Refund agent must call `lookup_order` before answering" → `fast-`
- "Response must be valid JSON containing `order_id`" → `fast-`
- "Never reveals another customer's data, however asked" → `safety-`
- "What is our return window?" (policy has one right answer) → `golden-`
- "Explains a declined refund clearly and empathetically" → `judged-`

## 3. When to add a case vs. a new file vs. a new tier

1. **New case, existing suite** — same tier, fits the file's theme → append to `eval_cases`. IDs are
   `<topic>-<behavior>-<nn>`, kebab-case, stable forever (they're join keys in the GCS results — never
   reuse or rename a shipped ID; retire by deletion).
2. **New file, same tier** — same metrics, new theme, or a file passing ~50 cases (reviewability
   split): `fast-tool-edge-cases.json`.
3. **New tier** — the case needs a *different metric suite* → new prefix + config template + one
   mapping line. Rare by design.

## 4. Keeping N agents consistent (scalability)

- **Templates + registered mapping:** repo-root `eval-templates/eval_config.<tier>.yaml` are the
  canonical suites, and `tools/eval_tiers.py` is the versioned prefix→profile-ID→requirements
  registry. A new agent copies the tiers it needs; **adding an agent requires no platform code changes
  when it follows the registered conventions**. The lint flags an agent config that drifts from its
  template unless it carries an explicit `# custom-suite: <reason>` header — divergence is allowed,
  but visible.
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

## 6. Judging paradigms — which we use where

- **Pointwise (absolute score vs fixed bar) — the v1 gate.** A merge gate requires a fixed bar, and
  only absolute scores provide one; `min_mean`/`min_case` thresholds are pointwise machinery. Known
  noise in absolute judge scores is mitigated by: generous initial thresholds (block regressions, not
  variance), persisted per-case rationale, and trend tracking by SHA.
- **Rubric-based — how pointwise stays trustworthy.** The `safety-` tier judge is rubric pass/fail
  (refusal criteria), which is far less noisy than a 1–5. Google's adaptive/per-prompt rubrics have
  live schema hooks we validated: `EvalCase.rubric_groups` (per-prompt criteria) and
  `rubric_verdicts` in results (present in the fixture, null today). They adopt with zero structural
  change when the backend enables the rubric premades (registry watch detects).
- **Pairwise (A vs B) — deliberately NOT in the v1 gate; the Phase-4 endgame.** Schema already
  supports it (`responses[]` is a candidate list, `win_rates` + `candidate_names` in results). Two
  designated uses: (1) forced model migrations — "which model answers our cases better" is an A/B
  decision, pairwise's strength; (2) **baseline-relative gating** — judge compares each new response
  against the last main run's response per case, gate on win-rate: pairwise reliability aimed at
  pointwise's noise, once baselines exist to compare against.

## 7. CI ordering & cost control

Run tiers cheapest-first, fail fast: `fast` → `judged` → `golden`/`grounded`/`safety` → `multiturn`.
Per-tier `--concurrency` on generate bounds parallel judge/model pressure. The judged tiers run only
on merge to main and on-demand — never per-commit (arch §8 rule 4). Every run's per-tier scores land
in the same GCS results layout regardless of tier, so the UI and trends need no tier-specific code.
