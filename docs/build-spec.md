# Agent Eval Platform — Build Spec (Handoff for Coding Agent)

**Companion to:** `agent-eval-architecture.md` (read it first for the why; this doc is the how).
**Everything marked ✅ was empirically verified on 2026-08-15** by installing the real packages in a clean Ubuntu 24 / Python 3.12 sandbox and running/inspecting them (many re-verified on macOS arm64 2026-08-15/16). Items marked 🔶 require live Google credentials and must be verified in Phase 1.

> **REV 3 (2026-08-16, owner decision — READ FIRST):** agents deploy to **Vertex AI Agent Platform
> (Agent Runtime)** day one, project `repp-501700`; evals run against the deployed revision via Agent
> Platform APIs; model access is **Vertex-only** (`GOOGLE_GENAI_USE_VERTEXAI=true` — scaffold default;
> no `GEMINI_API_KEY` anywhere). Consequences for this doc:
> - Scaffold agents with `-d agent_runtime` (verified flag: `--deployment-target [agent_runtime|cloud_run|gke|none]`), not `--prototype`.
> - §5's CI/on-demand flows: replace local `adk api_server` hosting with deploy → eval-against-revision.
> - §8's Runner: thin orchestrator, one image for all agents, no per-SHA agent baking. §12's two-process
>   design becomes the FALLBACK, not the default.
> - **Blocking spike before CI YAML** (source-verified 2026-08-16): in 1.3.1 only `eval submit`
>   (experimental) knows Agent Engine resource names — `generate --url` speaks the 4-route ADK HTTP
>   surface only. Deploy one agent, try `eval submit --resource-name` AND `generate --url <endpoint>`;
>   the winner is the CI execution step. `grade` and `tools/eval_gate.py` are unchanged either way.
>
> **REV 4 (2026-08-17, external review adopted — READ THIS OVER REV 3's SEQUENCE):**
> - **Production sequence inverted: eval-then-deploy.** CI checks out the exact Git SHA, runs the
>   agent FROM SOURCE inside the CI job (the §12-validated local path — `generate` spawning the local
>   server), evaluates the complete suite, gates, and **deploys only on pass** (branch protection /
>   pipeline gating makes it binding). The rev-3 spike result (`generate --url <runtime>/api`) is now
>   the **Apex Backoffice on-demand mechanism**, not the CI mechanism.
> - **Apex** (the internal UI's real name) resolves the deployed revision → its source SHA → loads the
>   suite at THAT SHA (never mixed with main) → runs cloud-supported metrics → reports repo-local
>   custom metrics as "not run" and the run as **partial**. CI is the authoritative complete-suite
>   gate. **The credential-bearing Runner never executes arbitrary repository Python.**
> - **Gate completeness rules added** (see §6): verify generated case IDs match the authored set;
>   verify every selected metric executed; correlate cases by ID, never by result-list order;
>   infrastructure errors reported distinct from quality failures.
> - **Judged gating softened**: participates day one, promoted to blocking on baseline stability
>   evidence (supersedes blocking-day-one). Red-team/safety coverage moves into Phase 0–1.
> - New Phase 4 stretch: authoring supported eval cases in Apex → GitHub PR via CODEOWNERS.

---

## 1. Verified Environment

```bash
pip install google-adk==2.6.3 google-agents-cli==1.3.1
```
- ✅ PyPI package name is **`google-agents-cli`** (NOT `agents-cli`); binary installed is `agents-cli`, v1.3.1.
- ✅ `adk` binary v2.6.3 installs from `google-adk`.
- ✅ Python 3.12 works. agents-cli docs recommend 3.11+.
- Pin both in every surface (local pyproject via `agents-cli install`/uv, CI, Runner image).

## 2. Empirical Findings That Override the Design Doc's Assumptions

| # | Finding | Consequence for the build |
|---|---|---|
| E1 | ✅ `agents-cli eval grade` **never exits non-zero for low scores** — only for operational errors (`click.ClickException`: no traces, parse failure, evaluation exception). No threshold flag exists in 1.3.1. | **You must build the CI gate**: a small script that parses grade results JSON and exits 1 when thresholds are unmet. Spec in §6. |
| E2 | ✅ `grade` internally calls `vertexai.Client(project, location).evals.evaluate(dataset=EvaluationDataset, metrics=[...])` — the Vertex AI Gen AI evals SDK. | The Runner Service may call this SDK directly instead of shelling out; identical scoring guaranteed. Note ✅ observed deprecation warning: `vertexai.Client` → `agentplatform.Client` — expect churn; wrap client creation in one function. |
| E3 | ✅ Even when **all metrics are local custom functions**, `vertexai.Client()` init **requires Application Default Credentials** (`DefaultCredentialsError` without them). There is no credential-free grade path in 1.3.1. | Every surface running `grade` needs Google creds — including local dev fast tier. For a credential-free deterministic gate, use ADK-level `adk eval` fast metrics (🔶 verify offline behavior) or run the gate script over generate traces directly. |
| E4 | ✅ `eval generate` runs the agent via a **local HTTP server** (project's `fast_api_app.py` if present, else `adk api_server`), cases run **in parallel**, and `--url` lets it target an **already-running/deployed agent**. | Two Runner Service architectures are now possible: (A) bake agent code into image and run locally (design default), or (B) `generate --url <deployed-agent>` against a running instance. Choose A now; B becomes attractive with Agent Engine. |
| E5 | ✅ Grade artifacts: `--output` dir (default `artifacts/grade_results/`) gets `results_<YYYYMMDD_HHMMSS>.json` **and** `results_<ts>.html`. The JSON is `EvaluationResult.model_dump()` plus `metadata.dataset` rows. | GCS upload = copy this directory; UI "run detail" can even iframe/link the HTML. Gate script parses the JSON. |
| E6 | ✅ Trace/dataset schema (vertexai `EvalCase`): fields include `eval_case_id`, `prompt` (genai Content: `{role, parts:[{text}]}`), `responses` (list of `{response: Content}` candidates), `reference` (`{response: Content}`), `conversation_history`, `intermediate_events`, `agent_data` (turns), `rubric_groups`, `system_instruction`. A hand-written trace with `prompt` + `responses[0].response` loads successfully. | Datasets/traces the coding agent writes must match this. My first guess (flat `response`) failed parse — use `responses[]`. |
| E7 | ✅ Scaffold (`agents-cli create X --prototype --yes`, offline-capable) actually produces: `app/{__init__,agent,fast_api_app}.py`, `app/app_utils/{a2a,services,typing}.py`, `tests/eval/{eval_config.yaml,response_quality.py,datasets/basic-dataset.json}`, `tests/{integration,unit}/`, `Dockerfile`, `.env(.example)`, `GEMINI.md`, `agents-cli-manifest.yaml`, `pyproject.toml`. | Slightly different from the docs-site tutorial (config sits at `tests/eval/eval_config.yaml`, plus a `response_quality.py` custom-metric file and a `Dockerfile`). Use this real layout. |
| E8 | ✅ `eval_config.yaml` real format: `metrics_to_run: [names]` + `custom_metrics:` where each entry has `name` and either `custom_function_file: file.py` or inline `custom_function: |` python defining `evaluate(instance) -> {'score': ...}`. Default scaffold metric `custom_response_quality` is a **local LLM-as-judge** via google-genai (ADC or `GEMINI_API_KEY`). | Tiering = two config files: `eval_config.fast.yaml` (deterministic custom metrics) and `eval_config.judged.yaml` (judge metrics). Pass via `grade --config`. |
| E9 | ✅ `grade` flags: `--traces PATH`, `--output DIR`, `--metrics a,b`, `--config FILE`, `--project`, `--region` (default `global`). `generate` flags include `--dataset`, `--output`, `--url`, `--app-name`, **`--concurrency N`** (default min(32, cores)) and **`-H/--header`** (repeatable custom headers, "overrides auto-detected auth" — generate can authenticate to deployed URLs). ✅ `adk eval` 2.6.3 contract matches the docs (module path + evalset paths, `file.json:eval_1,eval_2` selection). | Commands in the architecture doc are correct as written, modulo the package name fix. `--concurrency` is the knob for judge-cost/parallelism control per run. |
| E10 | ✅ Full eval command surface present in 1.3.1: `generate, grade, analyze, compare, dataset (synthesize), metric (list), optimize, run, submit, results`. | ✅ RESOLVED (2026-08-15, help text + docs): `eval submit` = experimental E2E **cloud-side** eval on Vertex AI Eval Service (`--resource-name` Agent Engine reasoningEngine, `--dataset`, `--dest gs://`); `eval results` polls a submitted run. Both experimental = the future Agent Engine path (arch §12); no impact on our design. |

### 2a. LIVE findings (2026-08-16, project repp-501700, gemini-3.7-flash via Vertex ADC) — closes Spikes A+B

| # | Finding | Consequence |
|---|---|---|
| L1 | ✅ End-to-end generate→grade works live on Vertex ADC: real model calls, scaffold LLM-judge (`custom_response_quality` via google-genai) scored them, `results_<ts>.json` + `.html` written exactly per E5. | ADC-only path proven — no API key anywhere. Grade-with-judge-via-ADC ✅ (spike a's local-judge half). |
| L2 | ⚠️ **`generate` exits 0 on PARTIAL success** — 2/3 cases succeeded → exit 0, artifact "contains only the 2 successful case(s); 1 failed case(s) were dropped" (printed, not recorded). Grade then sees only survivors: `num_cases_total: 2`. | **Gate requirement added**: `eval_gate.py` must take the dataset's expected case count and FAIL when graded cases < expected — otherwise CI can go green while cases silently drop. (0/3 → exit 1 + no artifact, verified earlier.) |
| L3 | ✅ **Results JSON schema captured** (fixture: `docs/fixtures/grade_results_fixture.json`): top-level `eval_case_results[] {eval_case_index, response_candidate_results[] {response_index, metric_results: {<name>: {metric_name, score, explanation, rubric_verdicts, raw_output, error_message}}}}`, `summary_metrics[] {metric_name, num_cases_total, num_cases_valid, num_cases_error, mean_score, stdev_score, pass_rate}`, `evaluation_dataset[]`, `metadata {candidate_names, creation_timestamp, dataset}`. | §11's top unknown CLOSED — write the gate parser against the fixture. Judge `explanation` is right there per case (persist it, per arch §8 rule 5). |
| L4 | ✅ Live trace artifact (fixture: `docs/fixtures/traces_fixture.json`): case keys `prompt, responses, eval_case_id, agent_data`; **no usage/token fields anywhere; `intermediate_events` absent**. | G1 confirmed on live data: tokens are NOT in eval artifacts. OTel/Cloud Trace (or BQ Agent Analytics `agent_events`) is the only token source. |
| L5 | GCP setup gotchas hit live: browser consent needs the Cloud checkbox ticked; `gcloud auth login` (CLI) is separate from ADC; billing account existed but was NOT linked to the project (`gcloud billing projects link` fixed it); API enablement propagates unevenly for ~10 min (parallel cases can 403 on stale frontends while others pass). | Onboarding doc material for every dev + the CI service-account runway. Transient 403s right after enablement are retryable, not real failures. |
| L6 | ✅ **SPIKE D DECIDED — PATH B WINS.** `agents-cli deploy -d agent_runtime` (no manifest edit needed; `--min-instances 0` = scale-to-zero) deployed to Agent Runtime in ~4 min. The deployment **exposes the full ADK api_server surface under `<resource>/api`**: `https://us-central1-aiplatform.googleapis.com/reasoningEngines/v1/projects/<num>/locations/<region>/reasoningEngines/<id>/api` → `/health`, `/list-apps`, `/apps/app/app-info` all 200. `eval generate --url <that>/api --app-name app` ran **3/3 cases with AUTO-DETECTED auth** (ADC bearer attached automatically — no `-H` needed). | **CI execution step = deploy → `generate --url <runtime>/api` → `grade` → `eval_gate.py`.** Every link verified live. The Runner needs only the resource name from the deploy manifest to construct the URL. Base URL WITHOUT `/api` 404s — the `/api` suffix is load-bearing. |
| L8 | ✅ **Forced-failure check passed**: corrupting the `capital_lookup` reference dropped `custom_response_quality` mean 5.0 → 3.67 while grade exited 0 (3/3 cases valid). | Low scores never fail the exit code — end-to-end proof of the gate premise. Gate fixture pair: healthy run 5.0 vs corrupted run 3.67 (`artifacts/grade_badref/`). |
| L9 | ✅ **Built-in Vertex judge metric works live**: `grade --metrics final_response_quality --region us-central1` against the runtime traces → exit 0, results saved. This is the Gen AI Eval Service path (not the scaffold's local custom judge). | Judged tier's real scoring path verified. Metric naming note: service metrics use names like `final_response_quality` (see `eval metric list`), distinct from ADK-level criteria names. |
| L10 | ⚠️/✅ Telemetry split finding: **token metrics CONFIRMED flowing** — Cloud Monitoring `aiplatform.googleapis.com/publisher/online_serving/token_count` shows live series from today's runs (latest point 8,195 tokens; `consumed_token_throughput` 16,271) — this is what the console dashboards chart. **Cloud Trace spans**, separately, showed 0 traces via the v1 API (2h window) — the trace/waterfall destination is unconfirmed. **Tool calls need neither**: they're captured per eval case in the generate artifacts (`agent_data` turns, name/args), gradeable and stored in GCS. | Tokens: per-case = impossible (not in artifacts); per-project/model = confirmed live. Cheap upgrade paths if per-run rollups are ever wanted: (1) Monitoring time-window query at manifest-write (~20 lines, approximate), (2) BQ Agent Analytics (exact per-event). Phase-1 item narrows to: confirm the console observability tab renders these metrics; investigate Trace only if span waterfalls are wanted for debugging. |
| L11 | ✅ **Per-run/per-case token surfacing is buildable with shipped components** (source-verified): installed ADK includes `google/adk/plugins/bigquery_agent_analytics_plugin.py`; its BQ event schema has `session_id`, `invocation_id`, `trace_id`, and content incl. `usage_metadata` ("detailed token counts"). `scaffold enhance --bq-analytics` wires it in. Eval traffic is attributable: hardcoded `eval-cli-user` + one session per case (§12 protocol). | **Owner decision (final, 2026-08-16): CI has ZERO token code** — CI is regression-gate only. **The UI backend computes tokens lazily at view time** for any run (query key = manifest `started_at/finished_at` window + `eval-cli-user`; GROUP BY session_id for per-case; window = start→now for a live counter on in-progress on-demand runs). Works retroactively for CI runs opened in the UI. Enable `--bq-analytics` on eval deployments in Phase 0/3; Monitoring window query (L10) is the approximate fallback until then. |
| L12 | ⚠️ **Premade-metric availability (live-tested 2026-08-16, us-central1 AND global):** WORKING — `final_response_quality` (mean 5.0 on healthy run), `final_response_match_v2` (3/3 valid, sensible partial-match scores), local `custom_function` metrics. **REJECTED BY BACKEND** (`400 Unsupported predefined metric`, both regions) — `grounding_v2`, `hallucination_v2`, `safety_v3`, `instruction_following_v2`, `tool_use_quality_v2`. The CLI's 17-metric registry is ahead of the service rollout (consistent with submit's 404). `multi_turn_*` untested. | Tier configs ship with the working set: judged = `final_response_quality` + local judges; golden = `final_response_match`. `grounded-`/`safety-` tiers are defined-but-dormant until the backend lights up — the `eval-metrics.snapshot` registry watch is the detection mechanism (re-test metric availability on every toolchain bump). |
| L13 | 🚨 **Errored cases are EXCLUDED from mean/pass_rate.** Mixed-reference dataset graded with `final_response_match`: 2/3 cases errored, and the result reports `mean=1.0, pass_rate=1.0` computed over the single valid case. Exit 0. A dataset can error on every hard case and look perfect. | **Hard gate rule: `eval_gate.py` fails the run when `num_cases_error > 0` or `num_cases_valid < dataset case count`** (subsumes L2's dropped-case rule at the grade layer). Surface per-case `error_message` in the gate table + UI. This rule is not optional. |
| L7 | ❌ **PATH A (`eval submit`) is non-functional in 1.3.1 — two independent failures.** (1) Sparse-reference datasets crash the SDK's dataset→dataframe conversion (`CandidateResponse(text=nan)` pydantic error for cases lacking `reference`) — review-gap **G6 manifesting live**. (2) With an all-reference dataset it reaches the backend, which returns **`404 Method not found`** — the cloud-side evaluation-run API isn't available (us-central1, 2026-08-16; preview/rollout-gated). | `eval submit`/`results` are shelved: re-test on future agents-cli/backend releases, gate nothing on them. G6's dataset-authoring rule (reference-based usage ⇒ `reference` on every case) is mandatory. Path B is the only working deployed-eval path today. |

## 3. Repo Structure To Build (monorepo)

> **Eval taxonomy is specified in `eval-conventions.md`** (tier prefixes → Google premade metric
> suites, live-verified metric registry, lint rules, template inheritance, scaling lanes). The
> `fast-*`/`judged-*` split below is the minimal Phase-0 subset of that taxonomy; build
> `tools/eval_tiers.py` + `tools/eval_lint.py` per that doc.

```
repo/
  agents/<agent_name>/          # one scaffold per agent: agents-cli create <name> --prototype --yes
    app/agent.py                # SET model= EXPLICITLY (e.g. Gemini(model="gemini-2.5-flash"))
    tests/eval/
      datasets/fast-*.json      # tier by filename prefix
      datasets/judged-*.json
      eval_config.fast.yaml     # deterministic/local custom metrics + thresholds block (§6)
      eval_config.judged.yaml   # judge metrics (custom_response_quality etc.)
      response_quality.py
  tools/eval_gate.py            # §6 — the one custom script
  pipelines/azure-pipelines.yml # §7
  services/eval-runner/         # §8 FastAPI service (Dockerfile FROM the agent scaffold pattern)
  ui/                           # §9 — three screens only
```

## 4. Credential Matrix (build-time truth)

| Operation | Needs | Verified |
|---|---|---|
| `agents-cli create`, scaffold, dataset editing | nothing | ✅ ran offline |
| `eval generate` | model access via **Vertex ADC** (`GOOGLE_GENAI_USE_VERTEXAI=true`, `GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION` — the scaffold `.env` default; rev 3 forbids AI Studio keys) | 🔶 needs live ADC |
| `agents-cli deploy` (agent_runtime) / `eval submit` / `eval results` | ADC + project + Agent Platform APIs enabled | 🔶 rev-3 blocking spike |
| `eval grade` — any metrics, even all-local | GCP ADC (client init hard-requires it) | ✅ proven (E3) |
| `eval grade` — built-in/judge metrics | ADC + `GOOGLE_CLOUD_PROJECT` (+ `--region`, default `global`) | 🔶 live |
| `adk eval` fast metrics (trajectory, response_match) | credential-free — adk.dev docs explicitly state these run locally, "suitable for frequent automated checks" | ✅ docs-confirmed 2026-08-15 (30s live sanity run still wise in Phase 0) |
| Cloud Trace via `--otel_to_cloud` | ADC + project | 🔶 live |
| gate script over grade JSON | nothing | ✅ by construction |

## 5. The Three Flows (final commands)

**Local:** `agents-cli playground` → edit → `agents-cli eval generate --dataset tests/eval/datasets/fast-smoke.json --output artifacts/traces/` → `agents-cli eval grade --traces artifacts/traces --config tests/eval/eval_config.fast.yaml` → `python tools/eval_gate.py artifacts/grade_results/ --thresholds tests/eval/eval_config.fast.yaml` → `eval analyze` / `eval compare` as needed → PR.

**CI (rev 4 — the authoritative gate, path-filtered per agent):** checkout exact SHA → install pinned
packages + the agent's deps → auth to GCP (WIF) → per tier: `eval generate` (agent runs FROM SOURCE in
the job via the local server path; `--session_service_uri memory://` if self-hosting the server) →
`eval grade --config <tier>` → `tools/eval_gate.py` with completeness checks (§6) — fail on low score,
errored case, missing case, or missing metric; infra errors distinct → publish artifacts +
`gsutil rsync` to `gs://<bucket>/results/<agent>/<git_sha>/<run_id>/` + `run.json` manifest →
**`agents-cli deploy` ONLY on pass** (record `{agent, resource_name/revision, git_sha}`) → branch
protection/pipeline gating makes the gate binding. Custom `custom_function` metrics run here (CI
already executes repo code). No per-merge Runner image build.

**Apex on-demand (rev 4):** `POST /agents/{a}/runs {tier}` → Runner (thin orchestrator) resolves the
agent's deployed revision from deploy manifests → resolves that deployment's git SHA → pulls datasets
+ configs from GitHub Contents API **at that SHA (never main-mixed)** → worker runs
`eval generate --url <runtime>/api` (rev-3 spike-validated) → `eval grade` with **cloud-supported
metrics only** — repo-local custom metrics reported as `not_run`, run marked **`partial`** → gate for
status → rsync results + manifest to the same GCS layout → Apex polls `GET /runs/{id}`. The Runner
never executes repository Python.

## 5b. Results Store — GCS layout and file schemas (consolidated, rev 4)

**One bucket is the system of record. CI and the Apex Runner write the IDENTICAL layout — that is why
one reader serves the CI view, the Apex run detail, and the interleaved history. Append-only: runs are
never mutated or deleted by the pipeline.**

```
gs://<project>-agent-evals/
  results/<agent>/index.json                    ← one JSON line per run — powers the history LIST
  results/<agent>/<source-version>/<run-id>/
    run.json                                    ← manifest: exactly what was evaluated
    gate_summary.json                           ← the verdict the CI log + Apex run-detail read
    grade_results/results_<ts>.json             ← Google's scores + judge rationale per case
    grade_results/results_<ts>.html             ← Google's human-readable report (free; link in UI)
    traces/traces_<ts>.json                     ← transcript per case (responses + tool calls)
```

Per-file schema status:

| File | Schema | Status |
|---|---|---|
| `grade_results/*.json` | Google's `EvaluationResult` dump | **captured live** — fixture at `docs/fixtures/grade_results_fixture.json`; gate parser written test-first against it |
| `traces/*.json` | `EvaluationDataset` with populated `responses`/`agent_data` | **captured live** — `docs/fixtures/traces_fixture.json` |
| `run.json` | ours | **specified**: `{agent, source_version, deployed_revision/runtime_identity (Apex runs), dataset_hash, config_hash, case_ids, metric_ids, model + toolchain + judge versions, trigger: ci|on_demand, requested_by, run_id, started_at, finished_at, status: running|passed|failed|error, completeness: complete|partial}` — finalized in Phase 0 with the gate |
| `gate_summary.json` | ours | **specified**: per-metric `{mean, floor_worst_case, bar, pass}`, per-case failures with judge rationale refs, completeness counts (authored vs generated vs graded vs errored), exit code + reason class (quality|infrastructure) |
| `index.json` | ours | one line per run: `{run_id, source_version, trigger, tier(s), status, completeness, started_at}` — appended by the writer after the run directory is complete |

Rules the writers obey: the run directory is written fully BEFORE the index line is appended (readers
never see a listed-but-incomplete run); CI runs and Apex runs differ only in `trigger`,
`deployed_revision` presence, and possible `partial` completeness; nothing else about the layout may
diverge between surfaces.

## 6. Gate Script Spec (`tools/eval_gate.py`) — the one custom component

Input: a grade_results dir (finds newest `results_*.json`) + thresholds. Thresholds live in a `thresholds:` block appended to each eval_config YAML (✅ VERIFIED 2026-08-15: grade tolerates the unknown key — config parsed, metric loaded, run proceeded to client init; the models are Pydantic `extra="allow"`. No sidecar needed):
```yaml
thresholds:
  agent_turn_count: {min_mean: 1.0}
  custom_response_quality: {min_mean: 0.7, min_case: 0.5}
```
Behavior: parse `EvaluationResult` JSON → per-metric mean + per-case scores → print table → exit 0 iff all thresholds met, exit 1 otherwise (exit 2 for infrastructure/parse/IO errors — **infrastructure failures must stay distinguishable from quality failures**). Also emit `gate_summary.json` (machine-readable, uploaded with the manifest) and optional `--junit out.xml` (one testcase per metric×dataset). Add unit tests with the fixture results JSON.

**Completeness rules (rev 4 — non-optional):**
1. Take the authored dataset as input and **verify generated/graded case IDs match the authored set** — fail on any missing case (closes L2's silent-drop).
2. **Verify every selected metric actually executed** — a metric absent from results is a failure, not a skip.
3. **Fail on any errored case** (`num_cases_error > 0` — closes L13's error-exclusion).
4. **Correlate cases by `eval_case_id`, never by result-list order** (results carry `eval_case_index`; join through the metadata dataset to IDs).
5. A missing or errored metric/case is NEVER a pass; dropped cases never disappear from completeness accounting.

**Watch item (2026-08-15):** ADK 2.7.0 added native threshold grading + crash-fails-eval to ADK's *own* eval engine (`adk eval`/`AgentEvaluator`) — the escape-hatch path only; agents-cli `grade` scores via the Vertex SDK and is unaffected. Google is converging on gate semantics, so on every agents-cli version bump, check whether `grade` gained a threshold flag / score-based exit codes — the release that does deletes most of this script. Do NOT switch CI to `adk eval` to get it early: different dataset format (`.evalset.json`), unverified exit-code contract, and it would split scoring across two engines (violates the one-engine invariant).

## 6a. How blocking actually works — the exit-code seam (read this if "grade succeeded, so how does it block?" is confusing)

Google's grading step ALWAYS succeeds — `grade` exits 0 even on terrible scores (E1, verified). It is
not the last step. The gate is one more step, ours, whose success is defined differently:

```
Step 1  generate       → exit 0   (agent ran, transcript written)
Step 2  grade          → exit 0   (Google scored it — always succeeds)
Step 3  eval_gate.py   → exit 0 ONLY if scores clear the bars     ← everything hinges here
```

The gate's core is nothing more than:

```python
results = read("results.json")            # Google's scores, written by step 2
if results.mean < min_mean or any(case < min_case) or completeness_violated:
    print(explanation_with_judge_rationale)
    sys.exit(1)          # ← THIS LINE IS THE ENTIRE BLOCKING MECHANISM
sys.exit(0)              # (exit 2 for infrastructure errors — distinct from quality failures)
```

CI runners are deliberately dumb: they know nothing about evals — they watch each step's exit code,
and any non-zero exit fails the job. Failed job → required status check goes red → GitHub disables
the merge button (block #1); failed job → the deploy job that `needs:` it never starts (block #2).
Wiring details: §7a.

Two different meanings of "successful" — keep them separate:

| | "Did the machinery work?" | "Did the agent measure up?" |
|---|---|---|
| Who answers | Google's `grade` (step 2) | **our gate** (step 3) |
| Exit code reflects | scoring completed | scores cleared the bars |
| Bad agent, working machinery | exit 0 | **exit 1 → blocked** |

Mental model: **`eval_gate.py` is pytest for eval scores.** Running pytest "succeeds" as a program even
when tests fail — but it exits 1 on failed assertions, and that is what turns CI red. Our gate asserts
`score >= bar` (plus the §6 completeness rules) instead of `x == y`. It exists as a component only
because Google's tool has no opinion about thresholds; if a future agents-cli grows a `--fail-below`
flag (watch item, §6), most of this script evaporates.

## 7. Azure Pipeline Skeleton

Triggers/connection per architecture doc §4.2. Steps: `UsePythonVersion@0 (3.12)` → `pip install google-adk==2.6.3 google-agents-cli==1.3.1` → `DownloadSecureFile` SA key + `export GOOGLE_APPLICATION_CREDENTIALS` → bash loop over `tests/eval/datasets/fast-*.json` (generate→grade→gate) → same for `judged-*` with `continueOnError: true` → `PublishTestResults@2` on gate JUnit → `PublishPipelineArtifact` grade_results → `gsutil -m rsync -r` to GCS → Docker build/push runner image. Matrix or per-agent path-filtered pipelines as agents grow.

## 7a. GitHub Actions Skeleton (rev 4.1 — the primary CI shape; §7's ADO skeleton is the fallback port)

One workflow, two triggers, two block points. The only custom component in the chain is the gate's
exit code — everything else is native CI machinery:

```yaml
# .github/workflows/quality-gate.yml
on:
  pull_request:                     # BLOCK #1: merge — via required status check
  push: { branches: [main] }        # BLOCK #2: deploy — via job dependency

jobs:
  checks:                           # PR + main: lint & gate unit tests, credential-free
    steps: [checkout, install pinned toolchain, run eval_lint, run gate unit tests]
  evals:
    needs: checks
    steps:
      - checkout                    # exact code version under review
      - google-github-actions/auth  # WIF, keyless
      - install pinned toolchain + agent deps
      - per tier (PR: fast only · main: full suite):
          agents-cli eval generate …   # agent runs FROM SOURCE in this job
          agents-cli eval grade --config eval_config.<tier>.yaml
          python tools/eval_gate.py …  # exits 1 on quality fail / 2 on infra → job dies here
      - publish artifacts + rsync results to GCS + write run.json/index.json
  deploy:
    needs: evals                    # ← deploy-block: never starts if evals failed
    if: github.ref == 'refs/heads/main'
    steps: [agents-cli deploy -d agent_runtime --min-instances 0, record revision in manifest]
```

Enforcement wiring (config, not code):
- **Branch protection / ruleset on `main`**: require the `evals` check → GitHub disables the merge
  button on red. This is the merge-block; it is server-side and cannot be bypassed by contributors.
- **`needs: evals`** is the deploy-block: a failed gate step fails the job, and GHA never starts a
  dependent job. No policy engine, no custom blocker.
- Optional hardening: a GitHub `production` Environment on the deploy job (approvals, wait timers).
- Failure UX: gate emits JUnit (`--junit`) → per-metric red annotations on the PR; `gate_summary.json`
  ships with artifacts for Apex.

Phase-1 acceptance = demonstrate both blocks live: a deliberately broken agent greys out the merge
button; a good merge auto-deploys. (ADO equivalents if the org requires: branch policies + stage
dependencies — see §7.)

## 8. Runner Service Spec (FastAPI, Cloud Run)

Endpoints: `GET /agents` · `GET /agents/{a}/instructions` (Contents API → app/agent.py raw + best-effort `instruction=` extraction + SHA) · `GET /agents/{a}/datasets` · `POST /agents/{a}/runs {tier: fast|judged|all}` → 202 `{run_id}` · `GET /runs/{run_id}` (includes lazily-computed `tokens` per L11) · `GET /runs?agent=…` (reads the per-agent `index.json`, not a GCS prefix scan — the Runner appends one line to `results/<agent>/index.json` on every run-write; storage decision: GCS stays the system of record for v1, Firestore/BQ-external-tables are additive upgrade paths per arch §15.4).
Rules (rev 4): per-agent run lock; Cloud Run service identity for GCP; GitHub App token (contents:read) for GitHub; **no agent code in the image and NO execution of repository Python** — the worker targets the deployed Agent Runtime revision via `generate --url <runtime>/api` and runs cloud-supported metrics only (repo-local custom metrics → `not_run`, run → `partial`); datasets/configs loaded at the deployed revision's SHA, never main-mixed; one small orchestrator image serves all agents; background execution via Cloud Tasks or asyncio worker; every result write carries full lineage `{agent, source_sha, deployed_revision/runtime_identity, dataset_hash, config_hash, case+metric identities, model/toolchain/judge/metric versions, trigger, requested_by, run_id, completeness: complete|partial}`.

## 9. UI (three screens, nothing else)

1. Agents list → run trigger (tier picker) + live status (poll).
2. Instructions view: raw agent.py + extracted instruction, SHA badge, "view on GitHub" link. Read-only.
3. Run history: list from GCS manifests (CI + on-demand interleaved); detail = gate summary, per-metric/per-case scores, judge rationale from results JSON, **token consumption (live counter while running via the L11 BQ query; per-run total + per-case breakdown after)**, link to grade HTML, Agent Platform console deep link.
No dashboards, no editing, no chat.

## 10. Build Order & Acceptance

| Phase | Build | Done when |
|---|---|---|
| 0 — Developer loop | Scaffold 1 agent **with `-d agent_runtime`**; tier configs + example datasets for **fast, safety/red-team, judged, golden**; `tools/eval_tiers.py` + `eval_lint.py` + `eval_gate.py` (test-first vs fixtures, incl. rev-4 completeness rules); one-command local dev experience | local generate→grade→gate passes; forced-fail exits 1; lint rejects a golden fixture missing a reference; ~~rev-3 spike~~ ✅ already closed live (Spike D) |
| 1 — Authoritative CI gate | GHA workflow: checkout SHA → pinned install → full suite from source → completeness-checked gate → publish raw+normalized results → **deploy only on pass**; branch protection/pipeline gating | a bad change is blocked BEFORE deploy; a good merge auto-deploys; results in GCS with full lineage |
| 2 — Apex walking skeleton | Authenticated thin Runner: resolve deployed revision + its SHA → trigger/watch runs → score, rationale, errors, **completeness (partial/complete)**, lineage → CI + on-demand history → console links | full loop demonstrable: local → CI gate → deploy → Apex on-demand → history |
| 3 — Hardening | Concurrency + run isolation; retention/authorization/sanitization; expanded live-validated profiles; quota controls; token accounting; stable result normalization + artifact correlation | two simultaneous runs serialize cleanly; tokens per run/case in Apex |
| 4 — Authoring (stretch) | Create/edit supported native eval cases in Apex (prompt, rubric criteria, optional golden ref) bound to a base SHA, preserving unknown native fields → opens GitHub PR → CODEOWNERS review; no metric-config editing from Apex | a case authored in Apex lands on main through normal review |
| Later | Baseline-relative gates · pairwise evaluation · managed Python sandbox (unlocks custom metrics in Apex) · simulated users/multi-turn generation · prompt optimization · trend + cost analytics | — |
| 1 spikes | (a) grade with judge metrics live incl. `--region`; (b) `adk eval` credential-free sanity run (docs-confirmed 2026-08-15); (d) `--otel_to_cloud` spans visible incl. tokens **+ run_id→trace correlation design (mandatory — see review G1: trace artifact provably carries no tokens)**. ~~(c) thresholds key~~ ✅ closed offline; ~~(e) eval submit~~ ✅ closed via help/docs; ~~(f) ADO→GCP WIF~~ ✅ Google-documented end-to-end — now a Phase-2 config task | each remaining spike closes a 🔶 in this doc |
| 2 | ADO pipeline | merge runs both tiers; fast failure blocks; results in ADO artifacts + GCS |
| 3 | Runner Service | on-demand run for a SHA lands manifests in GCS; instructions endpoint returns file+SHA |
| 4 | UI | all three screens against live service |

## 12. Hosted-Agent Eval Protocol — Captured Live (✅ 2026-08-15)

Ran `adk api_server` (ADK 2.6.3) hosting the scaffolded agent, then `agents-cli eval generate --url http://localhost:8770 --app-name app` against it, and logged every HTTP call.

**The exact protocol `eval generate` speaks to a hosted agent (per eval case, cases in parallel):**
```
GET  /apps/{app_name}/app-info                       (once — discovery; NOT in adk.dev docs but
                                                      confirmed in 2.6.3 source: api_server.py:1257)
POST /apps/{app_name}/users/eval-cli-user/sessions   (per case — session create; user hardcoded "eval-cli-user")
POST /run_sse                                        (per case — streamed agent run)
```
Plus `GET /health` for liveness. That is the **entire required surface** — 4 routes.

**Full `adk api_server` route inventory (from /openapi.json, v2.6.3)** for reference: `/health`, `/version`, `/list-apps`, `/run`, `/run_sse`, `/apps/{app}/app-info`, sessions CRUD (`GET/POST/DELETE/PATCH /apps/{app}/users/{user}/sessions[/{id}]`), artifacts + versions endpoints, memory PATCH, `/agent-identity/finalize`. A minimal hosted agent needs only the 4 above; the rest is optional surface.

**Minimum hosted-agent server = just run `adk api_server <agent_dir> --port N`.** Do not hand-build it. Notes:
- ✅ The scaffold's `fast_api_app.py` wraps the same `get_fast_api_app()` but adds `google.auth.default()` + Cloud Logging **at import** — it won't even start without ADC. For local/dev hosting, `adk api_server` starts credential-free; creds are needed only when a run hits the model.
- ✅ Failure semantics under a broken agent: each case fails individually (`Agent returned an error: DefaultCredentialsError`), generate prints per-case errors, writes **no artifact when 0 cases succeed**, and **exits 1**. So a dead/misconfigured hosted agent fails CI loudly at the generate step, before grading.
- ⚠️ **CORRECTED (2026-08-15, 2.6.3 source):** the server does NOT default to in-memory storage — `--use_local_storage` defaults **true**, so sessions/artifacts persist in local `.adk` disk storage between runs. For throwaway eval sessions start the server with `--session_service_uri memory://` (and `--artifact_service_uri memory://`), or accept `.adk` accumulation in the Runner container. Eval sessions must not leak state across runs.
- ⚠️ `eval generate` requires running from the agents-cli project root (walks up for `agents-cli-manifest.yaml`, legacy fallback `pyproject.toml`) — the Runner worker must `cd` into the agent dir before shelling out.

**Status of this section under rev 3:** this two-process self-hosted design is now the **FALLBACK**, kept because it is fully validated. The default is the thin orchestrator targeting Agent Runtime deployments (§8). The protocol capture above remains directly relevant: if the deployed Agent Runtime endpoint exposes these 4 routes, path B (`generate --url`) works against it unchanged — that is exactly what the rev-3 blocking spike tests.

---

## 11. Known Unknowns for the Coding Agent (do not assume)

- ~~Exact `EvaluationResult` JSON field names~~ ✅ CLOSED (L3) — real fixture at `docs/fixtures/grade_results_fixture.json`; write the parser test-first against it.
- Whether judge metrics honor `GEMINI_API_KEY` without ADC on the agents-cli path (scaffold comment says yes for the local judge; E3 says client init still wants ADC — test both). Context (2026-08-15): at the **ADK layer**, docs confirm several judge criteria accept `GOOGLE_API_KEY`; only `safety_v1` + `multi_turn_*` hard-require a GCP project.
- agents-cli release cadence/breaking changes: **1.3.1 confirmed latest (2026-08-15; released 2026-08-04)** — re-run `--help` diffs on every bump. The E2 deprecation warning is a real mid-flight rename: `vertexai.Client` → `agentplatform.Client` warnings landed in google-cloud-aiplatform 1.159–1.162 (docs now branded "Gemini Enterprise Agent Platform"). Keep client creation wrapped in one function.
- **ADK 2.7.0 (2026-08-13) vs pinned 2.6.3**: 2.7.0 grades judge/ROUGE metrics against criterion thresholds and fails evals when the agent crashes before metrics — decide the pin at Phase 0 and freeze (agents-cli grade path unaffected).
- **Model pin**: scaffold default `gemini-3.6-flash` is GA but previous-generation; `gemini-3.7-flash` is the current recommended stable Flash (ai.google.dev, 2026-08-15). Either is a valid pin; pick the one prod will use and freeze it.
