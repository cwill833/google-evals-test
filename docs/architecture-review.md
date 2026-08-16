# Architecture Review — Pre-Handoff Validation

**Reviewed:** `agent-eval-architecture.md` (rev 2), `build-spec.md`, `live-validation-runbook.md`
**Date:** 2026-08-15
**Purpose:** find everything a build agent would have to *guess* about, before we hand it the spec.

**Verdict:** the architecture is sound — the shape (git as source of truth, one engine across three
surfaces, SHA-coupled results, GCS-first store) holds up, and `build-spec.md`'s empirical findings are
the strongest part of the package. What blocks handoff is not the design, it's that the docs are
**half-migrated from rev 1 (ADK-native) to rev 2 (agents-cli)**. A coding agent reading them today gets
contradictory instructions in five places and will silently pick one.

Fix B1–B5, answer G1–G5, then hand off.

---

## B — Blocking contradictions (agent will guess, and may guess wrong)

### B1. Tier mechanism is specified two different ways
- `arch §3` + `build-spec §3`: tier by **filename prefix** (`fast-*.json` / `judged-*.json`) in one flat
  `datasets/` dir, plus two config files.
- `arch §4.3.4`: "the tiers map directly to the `eval/fast|judged/` **folders** pulled from GitHub".
- `arch §8` example path: `eval/judged/test_config.json`.

**Resolution:** filename prefix is the rev-2 decision and matches the verified scaffold (E7). Delete the
folder language from §4.3.4 and §8.

### B2. Config file identity: `test_config.json` vs `eval_config.yaml`
`arch §3` and `§14` use `eval_config.yaml` (agents-cli). `arch §7 Phase 0`, `§8`, and `§15.5` still say
`test_config.json` (ADK). E8 empirically verified the YAML format (`metrics_to_run:` + `custom_metrics:`
with `custom_function_file` / inline `custom_function`) — that is the real one.

**Resolution:** purge `test_config.json` from rev-2 prose, or scope each mention explicitly to the
`adk eval` escape hatch. Note the two formats are **not** interchangeable: the ADK `criteria` dict in
§8's example has no equivalent in `eval_config.yaml`, and thresholds live in our own `thresholds:` block
(build-spec §6) — that gap is exactly why `eval_gate.py` exists.

### B3. `--eval_storage_uri` does not exist on the chosen runner
`arch §5` builds the `raw/` GCS prefix on `--eval_storage_uri gs://...` and claims it gives artifact
persistence "for free on every surface". But that is an **`adk eval` flag** (claim 12). The rev-2 runner
is `agents-cli eval generate` + `grade`, whose verified flags are `--output` / `--traces` writing to a
local dir (E5, E9). Under rev 2, `raw/` is unreachable and the "for free" claim is false — the real
mechanism is `gsutil rsync` of `artifacts/` (which build-spec §5/§7 already say).

**Resolution:** drop `raw/` from the GCS layout, or keep it and state that it is populated only when the
`adk eval` path is used. Uploading traces + grade_results under `results/` covers the need.

### B4. §7 Rollout Plan is un-migrated rev-1 text
It contradicts decisions made later in its own document: Phase 2 says results go to
"Firestore/BigQuery", but `§5` decides **GCS as system of record, BigQuery deferred**. Phase 0 says
"local `adk eval` working" and `test_config.json`. Phase 3 predates the §12 two-process finding.

**Resolution:** rewrite §7 against `build-spec §10` (which is current and has acceptance criteria), or
delete §7 and point at it. Two rollout plans is worse than one.

### B5. Every developer needs GCP creds on day one — the arch doc says otherwise
`arch §4.1` says "Judged tier locally requires GCP ADC", implying the fast tier is credential-free.
`build-spec E3` proves the opposite: `vertexai.Client()` init hard-requires ADC **even when all metrics
are local custom functions**. There is no credential-free `grade` path in 1.3.1.

This is the single largest **operational** consequence in the package and it isn't carried into the
design doc's onboarding, quota, or cost sections:
- Every dev needs a GCP project + `gcloud auth application-default login` before they can grade anything.
- `arch §6`'s "devs get a low-quota project" becomes mandatory, not a judged-tier nicety.
- The `adk eval` escape hatch is the only candidate credential-free deterministic gate — and it's still
  🔶 unverified (build-spec §4). Spike (b) in build-spec §10 is therefore higher priority than its
  ordering suggests; if it fails, offline local grading is simply not available and the docs should say so.

**Resolution:** update §4.1 and §6, and promote spike (b).

---

## G — Design gaps (unspecified; agent will invent something)

### G1. Nothing correlates `run_id` to a trace — so the token rollup has no join key
`§5a` says token rollups come "from trace/analytics data at manifest-write time", and `§5`'s
`EvalCaseResult` carries `trace_ref`. But no mechanism sets it. Worse, per `build-spec §12` the spans are
emitted by the **agent server process** (`adk api_server`), not by the `eval generate` process that knows
the `run_id`. Nothing described propagates the ID across that boundary.

**Needs specifying:** how `run_id` reaches the span (OTel resource attribute on server start — viable
because the Runner controls the server lifecycle; or a baggage header on `/run_sse`), and how the
manifest writer queries them back (Cloud Trace API filter + ingestion-delay retry). Without this, tokens,
`trace_ref`, and the UI's Cloud Trace deep link all quietly don't work.

**Cheaper alternative: ❌ REFUTED by source inspection (2026-08-15).** The eval-schema `Event` type
(`vertexai/_genai/types/evals.py`) has exactly four fields — `event_id`, `content`,
`creation_timestamp`, `author` — no `usage_metadata`, no token counts anywhere in the trace schema, and
`cmd_generate.py` never writes usage data. The generate artifact **cannot** carry tokens. Consequence:
the OTel/Cloud Trace route is the *only* path to token data, so the run_id↔trace correlation design
above is **mandatory, not optional** — or the design drops per-case tokens and settles for per-run
aggregates from Cloud Trace / BigQuery content logs. Decide before handoff.

### G2. One Runner image cannot hold N agents' dependencies
Each agent is a separate agents-cli scaffold with its **own `pyproject.toml`** (E7), but `§4.3` /
`build-spec §12` bake agent code into a single Cloud Run image tagged @ SHA. With more than one agent
that's one resolved dependency set for all of them — the first genuine version conflict between two
agents breaks the image, and `eval generate` additionally requires `cd`-ing into each agent's project
root (§12 note).

**Options:** per-agent image (tag `<agent>-<sha>`, cleanest, matches path-filtered CI); per-agent venv in
one image; or a declared monorepo constraint that all agents share one lockfile. Pick one now — it
changes the CI pipeline and the Runner's process model.

### G3. The 409 drift check has a self-inflicted outage window
`build-spec §5` has the Runner 409 when GitHub `main` ≠ image SHA. But the image is built *by* the
merge-to-main pipeline, so **every merge opens a window** — the length of a CI build — where all
on-demand runs for that agent fail. Rapid merges keep it open.

**Needs a fallback:** let the UI run at the image's SHA (labeled "running SHA abc123, main is at def456"),
which is arguably more correct anyway since the image *is* the agent code being evaluated. Reserve the
409 for a genuine mismatch the user opted into.

### G4. `EvalCaseResult.pass` is undefined
`§5` has `pass: bool` per case; `build-spec §6` computes per-metric **mean** plus `min_case`. Nothing
maps metric scores to a per-case boolean, and with multiple metrics per case the rule isn't obvious
(all-must-pass? weighted?). Either define it (recommend: all metrics with a `min_case` threshold must
meet it; metrics without one don't vote) or drop the field and let the UI render raw scores.

### G5. Run isolation against a shared agent server
`eval generate` runs cases **in parallel** (E4) against one `adk api_server` using **in-memory** sessions
(§12), with the user hardcoded `eval-cli-user`. The per-agent run lock (§8) lives in the Runner Service —
so it does not cover a CI job grading the same agent at the same time, since that's a different process
entirely. Low blast radius (sessions are per-case and throwaway), but state the isolation guarantee
explicitly, or move the lock somewhere both surfaces observe.

### G6. Most eval cases have no `reference` — metrics aren't uniformly applicable
Verified in the real scaffold: of 3 cases in `basic-dataset.json`, only `capital_lookup` has a
`reference` field. `greeting` and `weather_query` have `eval_case_id` + `prompt` only. That's why the
default config works — `custom_response_quality` (rubric judge) and `agent_turn_count` need no reference.

The moment a reference-based metric is added (`final_response_match_v2`, `response_match_score`), **2 of
3 cases become unscorable for that metric.** Neither doc says what happens then — skip, score 0, or
error — and a naive gate that averages across cases will silently drag the mean toward 0 and fail a
perfectly good agent.

**Needs:** a "metric not applicable to this case" concept in `eval_gate.py` (exclude from the mean, and
report the excluded count so it can't hide), plus a dataset-authoring rule that reference-based tiers
require `reference` on every case. This compounds G4 — it's the same missing definition of what "pass"
means at case granularity.

---

## R — Runbook corrections (found by running it)

### R1. ⚠️ Step 3's model instruction is wrong and would pin to a dying model
The runbook says *"scaffold default is a preview model — we pin"* and directs you to set
`MODEL = "gemini-2.5-flash"`. Both halves are now false:
- The real scaffold default is **`gemini-3.6-flash`** (`app/agent.py:25`) — no `-preview` suffix.
- `gemini-2.5-flash` is, per this package's own `arch §11`, **scheduled for shutdown 2026-10-16** — about
  two months out. Pinning validation to it means re-baselining every eval score before year end.

**Correction:** keep the scaffold default `gemini-3.6-flash` and pin it explicitly (the §11 rule is
"pin it", not "pin it to 2.5"). Confirm it's GA rather than preview before committing to it — the naming
convention implies GA, but that wasn't verified.

---

## S — Scope framing (not a defect, but name it)

**S1.** "Deployment is explicitly out of scope" (§ header) sits awkwardly against `build-spec §12`, which
requires `adk api_server` **running in Cloud Run** to evaluate anything on demand. That is deploying an
agent — just for eval, not for traffic. Call it "eval-only hosting" explicitly so nobody reads the deploy
phase as pre-solved, and so the Runner gets honest scaling/SLO expectations (cold starts land on eval
latency).

---

## V — Could not verify from here

**V1. ✅ CLOSED — re-verified independently on macOS 14 / arm64 / Python 3.12.11, offline, no credentials:**
- `pip install google-adk==2.6.3 google-agents-cli==1.3.1` resolves and installs clean. Binaries report
  `adk, version 2.6.3` and `agents-cli, version 1.3.1`. Package name `google-agents-cli` confirmed (E1/§1).
- **E7 confirmed exactly** — `agents-cli create test-agent --prototype --yes` produced the claimed layout
  file-for-file. The "not authenticated with Google Cloud" warning is **non-fatal** ("Continuing with
  template processing"), so scaffolding really is credential-free as build-spec §4 claims.
- **E8 confirmed exactly** — `eval_config.yaml` is `metrics_to_run:` + `custom_metrics:`, with both
  `custom_function_file:` and inline `custom_function:` forms present in the default scaffold.
- Transitive deps pull `google-cloud-aiplatform 1.164.0` (the `vertexai` SDK behind E2) and, notably,
  `litellm` + `openai` — worth knowing for Runner image size and supply-chain review.

**V1b. Deeper local re-verification (same session, installed 1.3.1 / 2.6.3 binaries + source):**
- **E1 source-confirmed**: every failure path in `cmd_grade.py` is an operational `ClickException`
  (no traces / parse failure / no cases / missing pyproject / eval exception → exit 1). No score-based
  exit exists in the code. Measured exit 1 on a real operational failure.
- **E2 deprecation warning reproduced verbatim**: `FutureWarning: The vertexai.Client class is
  deprecated. Please use agentplatform.Client instead.` at `cmd_grade.py:190`.
- **E3 re-confirmed on macOS**: `grade` with ONLY a local inline custom metric and no ADC fails at
  Vertex client init. No credential-free grade path — B5 stands.
- **Spike (c) CLOSED — `thresholds:` key is tolerated.** A config with `metrics_to_run` +
  `custom_metrics` + an extra `thresholds:` block parsed cleanly, loaded the metric, and proceeded all
  the way to client init. Root cause found: the Pydantic models use `extra="allow"`. build-spec §6 can
  keep thresholds in the eval_config YAML; no sidecar file needed.
- **`eval submit` UNKNOWN CLOSED**: experimental "Submit an E2E cloud-side evaluation run on Vertex AI
  Eval Service" — takes `--resource-name` (Agent Engine `reasoningEngines/...`), `--dataset`,
  `--dest gs://...`. Companion command **`eval results`** (fetch results of a completed cloud run) exists
  and is missing from build-spec E10's command list. Both experimental; neither changes our design —
  they're the future Agent Engine path (§12 of the arch doc).
- **`eval generate` has flags E9 missed**: `--concurrency` (default min(32, cores)) — relevant to G5,
  a real concurrency control exists — and `-H/--header` (repeatable, "Overrides auto-detected auth"),
  meaning generate can authenticate to a *deployed* agent URL; strengthens the E4-option-B path.
- **E6 schema confirmed field-for-field** against `vertexai/_genai/types/common.py:3345`, plus two
  fields E6 missed (`agent_info`, `user_scenario` — the latter is the user-simulation hook).
- **`adk eval` contract, `--eval_storage_uri`, `--otel_to_cloud`/`--trace_to_cloud`, `adk conformance`**
  all present in the installed 2.6.3 binary's help. Note `adk migrate` migrates *session databases only*
  — eval dataset migration is the agents-cli docs guide, not an ADK command.

Remaining caveat: the ADK 2.2.0 default-model changelog claim is still only attested by the original
session. Given ~bi-weekly ADK cadence, **re-run the `--help` diffs at build start**, not just design time.

---

## W — Live-web fact-check (2026-08-15, research agents against primary sources)

**W1. agents-cli — all 12 checked claims CONFIRMED** (github.com/google/agents-cli, PyPI,
google.github.io/agents-cli). Highlights beyond confirmation:
- **1.3.1 is the latest release** (2026-08-04; 1.3.0 same-day-patched) — the pin is current, and no
  post-1.3.1 release changes grade's no-gating behavior.
- **Google's own scaffolded CI/CD templates contain NO eval gating** — PR checks gate on
  `pytest tests/unit|integration`; staging runs Locust. `eval grade` appears nowhere in their shipped
  workflows. Consequence for `arch §4.2`: the instruction "mirror whatever mechanism the scaffold's
  CI/CD templates use" resolves to *nothing to mirror* — `tools/eval_gate.py` is not a fallback, it is
  the only mechanism. Remove the spike language; the question is answered.
- Official "Migrating Eval Datasets" guide confirmed (ADK `.evalset.json` → `EvaluationDataset`).
- Cloud Trace default-on confirmed; **content logs (GCS/BQ Agent Analytics) are opt-in** via
  `infra single-project` — run *after* deploy. Arch §5a's "provisioned" wording should note opt-in.
- v1.0.0 removed all RAG commands — ignore older third-party tutorials mentioning `infra datastore`.

**W2. Azure DevOps — all 5 checked claims CONFIRMED, 1 refuted, and the WIF spike is CLOSED:**
- **Claim 11 (ADO→GCP WIF) upgrades from "spike" to "officially documented".** Google's IAM doc
  *Workload identity federation with deployment pipelines* covers Azure DevOps end-to-end: the ARM
  WIF service connection's OIDC token → workload identity pool (attribute mapping
  `assertion.sub.split('/sc/')[1]`, condition on `assertion.oid`, token audience app ID documented) →
  SA impersonation via a credential-config file in pipeline YAML. Update `arch §6` and `§9 row 11`;
  the SA-key Secure File fallback remains valid but Google explicitly discourages keys.
- Claims 8, 9, 10 (GitHub App connection preferred; path-filter gotchas incl. case-sensitivity and
  no-variables-in-triggers; PublishTestResults@2 JUnit) all confirmed on current Microsoft Learn.
- **Refuted:** no Google-maintained ADO extension for GCP auth exists (community only) — plan on
  gcloud + credential-config files in plain script steps.
- **New gotcha:** since ADO sprint 227 an org/project setting can disable *implied* YAML CI triggers —
  always write an explicit `trigger:` block (we do) and check the org setting during Phase 2.

**W3. ADK — 11 of 15 checked claims CONFIRMED, 3 PARTIAL, and two disputes settled by local source:**
- **ADK 2.7.0 released 2026-08-13** — the docs' "latest = 2.6.3" was stale within a week; real cadence
  ~weekly; 1.x line at 1.38.0. 2.7.0 changes eval semantics (judge/ROUGE graded against thresholds,
  crash-fails-eval, CSV export, result persistence). Pin decision moved to Phase 0 (arch §11 updated).
- 2.0 breaking changes and the 2.2.0 default-model change ✅ confirmed against release notes.
- `adk eval` contract, `test_config.json` defaults (1.0/0.8), `AgentEvaluator` pytest path,
  `--eval_storage_uri` (on eval/web/api_server), `adk conformance` record/replay, `--otel_to_cloud`
  (≥1.17, GenAI semconv), Python-only eval, per-event `usage_metadata` ✅ all confirmed on adk.dev.
- **Judge auth softer than claimed**: several ADK judge criteria accept plain `GOOGLE_API_KEY`; only
  `safety_v1` + `multi_turn_*` hard-require a GCP project (arch §4.3 + claim row 5 updated). agents-cli
  `grade` still hard-requires ADC regardless (E3) — B5 unaffected.
- **Dispute 1 — `/apps/{app}/app-info`**: absent from adk.dev docs, but PRESENT in installed 2.6.3
  source (`api_server.py:1257`). Build-spec §12's live capture was right; docs gap only.
- **Dispute 2 — session storage**: web agent right, build-spec §12 was wrong. 2.6.3 source:
  `use_local_storage` defaults **true** → persistent local `.adk` storage, not in-memory. Runner must
  start the server with `--session_service_uri memory://` for throwaway eval sessions (spec corrected).
- Docs domain moved: google.github.io/adk-docs → **adk.dev**.

**W4. Gemini / Vertex / Agent Engine — all 12 checked claims CONFIRMED (2 with nuance):**
- `gemini-2.5-flash` retirement **2026-10-16 confirmed on the Vertex side**; nuance: the Gemini API
  (ai.google.dev) deprecations page still says "no shutdown date announced" for the same ID.
- `gemini-3.6-flash` is real and **GA — but already previous-generation**; current recommended stable
  Flash is **`gemini-3.7-flash`** (runbook step 0.3 corrected; R1 resolved).
- Free-tier AI Studio data-use-for-improvement ✅ (terms effective 2026-03-23); paid tier does not.
  AI Studio key ↔ real GCP project ✅ — the account-setup guidance in this session stands.
- `vertexai.Client` → `agentplatform.Client` migration is real and mid-flight (warnings landed in
  google-cloud-aiplatform 1.159–1.162; docs rebranded "Gemini Enterprise Agent Platform"). E2's
  wrap-the-client advice validated. Legacy `vertexai` generative modules were hard-removed 2026-06-24.
- Agent Engine claims (Sessions/Memory Bank GA + billing 2026-01-28, observability dashboard,
  Evaluation Layer + User Simulator, ~$0.0864/vCPU-hr) ✅; BigQuery Agent Analytics ✅ (BQ `agent_events`
  table **includes token usage** — a possible second source for per-run token rollups, relevant to G1);
  Cloud Trace GenAI spans ✅ (tokens as `gen_ai.usage.*` attributes, not a dedicated UI field).

---

## Fact-check verdict (2026-08-15, four research agents × primary sources + local source inspection)

**~45 load-bearing claims checked: zero fabrications found.** The package's empirical findings (E1–E10,
§12) survived both live-web checking and independent re-execution. Every discrepancy found was
*staleness* (2.7.0, gemini-3.7, session-storage default, docs domains) or *internal rev-1/rev-2
migration debt* (B1–B5) — the kind of drift the docs themselves predicted with their "re-verify at
implementation" caveats. Corrections have been applied to all three docs.

**V2.** The three sequence/context diagrams were not cross-checked against the prose in this pass. Do
that after B1–B4 land, since those fixes change what the diagrams should show.

---

## Doc hygiene

- `agent-eval-architecture.md` is missing **§10** and **§13** (jumps 9 → 11 → 12 → 14).
- `build-spec.md` orders sections **… 9, 10, 12, 11** — §12 is the most important recent finding and it's
  buried after the acceptance table.
- `build-spec §2` E-numbers are referenced from the arch doc but the arch doc never explains the E-prefix.

Cosmetic, but this is a handoff package — an agent that can't resolve a cross-reference will guess.

---

## Recommended order of work — status after the 2026-08-15 verification pass

1. ~~Fix B1–B5~~ ✅ **DONE** — all five contradictions corrected in the docs, informed by the live
   fact-check (gate mechanism resolved outright: Google's own CI templates contain no eval gating).
2. ~~Spikes (c), (e), (f)~~ ✅ **CLOSED** offline/via docs (thresholds tolerated; eval submit explained;
   WIF officially documented). ~~G1 cheap alternative~~ ❌ refuted at schema level.
3. **Run the remaining live runbook steps** (needs GEMINI_API_KEY then ADC): Spikes A, B, C + step 4 —
   produces the gate-parser fixture and closes `--otel_to_cloud` token visibility.
4. **Decide the design gaps: G1 (run_id↔trace correlation — now mandatory), G2 (per-agent images),
   G3 (409 window), G4+G6 (per-case pass semantics), G5 (shared-server isolation — note generate's
   `--concurrency` flag and the `.adk`-storage correction give the levers).** G2/G3 change pipeline
   shape — decide before YAML is written.
5. **Pin decisions at Phase 0**: ADK 2.6.3 vs 2.7.0 (eval-semantics changes in 2.7.0); model
   `gemini-3.7-flash` (current stable) vs scaffold's 3.6.
6. **Then hand off.** After this pass the three docs agree with each other and with the live toolchain.
