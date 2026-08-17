# Agent Evaluation Platform — Architecture & Design

**Status:** Draft for review
**Scope (rev 4):** Evaluation **plus deployment to Vertex AI Agent Platform** (Gemini Enterprise Agent Platform — the managed Agent Runtime, formerly Agent Engine). **Production sequence: local dev → CI evaluates the exact source SHA → gate → deploy only on pass → Apex Backoffice evaluates the deployed revision on demand.** Production *traffic* serving/monitoring remains a later phase.
**Stack decision (rev 3):** Google **agents-cli** layered on **ADK 2.x** as the toolchain; **Agent Platform as the runtime and eval surface** (project `repp-501700`); model access **Vertex-only** (`GOOGLE_GENAI_USE_VERTEXAI=true` — the scaffold default; no AI Studio keys anywhere); eval datasets versioned in GitHub; pipelines in Azure DevOps; observability via Cloud Trace (default-on) + Agent Platform console dashboards + BigQuery Agent Analytics.

> **Revision notes:** rev 1 = ADK-level tools directly (`adk eval`, `AgentEvaluator`) — still the escape hatch. Rev 2 adopted agents-cli as the interface. **Rev 3 (2026-08-16, owner decision): Agent Platform is the deployment target and eval API from day one** — this was previously §12's "future simplifier"; it is now the present. Key consequence: eval *scoring* already hit the Agent Platform eval API (grade → Vertex Gen AI Eval Service, build-spec E2); what rev 3 moves onto the platform is agent *inference/hosting*. The former G2 (multi-agent Runner image deps) and G3 (image↔main drift window) problems dissolve — the deployed revision is the thing under test.
>
> **Rev-3 blocking spike — ✅ RESOLVED LIVE (2026-08-16, on repp-501700): PATH B WINS.** The Agent
> Runtime deployment exposes the full ADK api_server surface at
> `https://<region>-aiplatform.googleapis.com/reasoningEngines/v1/<resource-name>/api` (the `/api`
> suffix is load-bearing; without it → 404). `eval generate --url <that> --app-name app` ran 3/3 cases
> with auto-detected ADC auth; `grade` scored the traces normally. **CI/Runner execution step =
> `deploy` → `generate --url <runtime>/api` → `grade` → `eval_gate.py`** — every link verified live.
> Path A (`eval submit`/`results`) is non-functional in 1.3.1: sparse-reference datasets crash the SDK
> (G6 live), and with references fixed the backend returns `404 Method not found` (API not rolled out).
> Shelved; re-test on future releases. Details: build-spec §2a L6/L7.
>
> **REV 4 (2026-08-17, external review adopted — supersedes rev 3's sequence):** the production
> sequence is **eval-then-deploy, not deploy-then-eval**: CI checks out the exact Git SHA, runs the
> agent *from source* in the CI job (the §12-validated local-execution path), evaluates the complete
> configured suite, and **deploys only on gate pass**. The on-demand surface is **Apex Backoffice**,
> which evaluates the *deployed runtime revision* via `generate --url <runtime>/api` (the rev-3 spike
> result — now the Apex mechanism, not the CI mechanism). Invariant reworded: **"One versioned
> evaluation suite and one grading contract across every surface."** Surfaces evaluate different
> targets (laptop → local agent; CI → source @ SHA pre-deploy; Apex → deployed revision) and every
> run records full lineage (source SHA, runtime identity, dataset/config hashes, model/toolchain/
> judge/metric versions, trigger, requester, run ID). **Results are comparable when source, dataset,
> configuration, models, metrics, and execution target match** — never claim automatic comparability
> across targets. Judged tier softened: participates day one, promoted to blocking on baseline
> stability evidence. New stretch goal: authoring supported eval cases in Apex → GitHub PR under
> CODEOWNERS. Security constraint: the credential-bearing Runner never executes arbitrary repo
> Python — Apex runs may initially execute cloud-supported metrics only, reporting local custom
> metrics as "not run" and the run as **partial**; CI remains the authoritative complete-suite gate.

---

## 1. Mental Model

One sentence: **Eval sets are versioned test artifacts that live in the agent's repo; three execution surfaces (local CLI, CI, on-demand UI) all run the same ADK eval engine against the same files; all results flow to one results store.**

Analogy to classic software testing:

| Classic testing concept | Agent eval equivalent (Google toolchain) |
|---|---|
| Test file / test suite | eval dataset JSON in `tests/eval/datasets/` (multiple per agent; agents-cli `EvaluationDataset` format — ADK `.evalset.json` migratable via the official migration guide) |
| Individual test case | Eval case (a recorded session with expected tool calls + reference response) |
| Assertions / thresholds | `eval_config.yaml` metrics configuration (trajectory / response-match / LLM-judged metrics; per-tier configs) |
| Test runner | `agents-cli eval generate` (produce responses) + `eval grade` (score) — ADK eval engine underneath; `adk eval` / `AgentEvaluator` remain the lower-level escape hatch |
| CI test report | JUnit XML → Azure DevOps Tests tab; scores → results store |
| Re-run a suite on demand | Eval Runner Service pulls sets from GitHub@main and executes |

Key invariants to architect around:

1. **GitHub `main` is the single source of truth** for both agent code and eval sets. No surface runs eval sets from anywhere else.
2. **One eval engine.** Local, CI, and on-demand all invoke ADK evaluation. No parallel scoring logic to drift.
3. **Agent code and eval sets are coupled at a git SHA.** ADK evaluation imports the agent module and runs it — so an eval run is only meaningful as (agent code @ SHA) × (eval set @ SHA) × (criteria). Every stored result must carry the SHA.
4. **Results are append-only and comparable.** CI runs and on-demand runs write to the same store with the same schema, so trends are one query regardless of who triggered the run.

---

## 2. Goals / Non-Goals

**Goals**
- Devs can run all eval sets for an agent locally with one command.
- Merge to `main` triggers CI (Azure DevOps, GitHub-hosted repo) that runs **all** eval sets for the changed agent and surfaces pass/fail per dataset.
- An internal UI can trigger a full eval run for any agent on demand, pulling the current eval sets from GitHub and running them via Google's eval tooling.
- Results (per-run, per-set, per-case scores) are persisted and queryable over time.

**Goals (rev 3/4)**
- Agents deploy to Vertex AI Agent Platform (Agent Runtime) — the same runtime production will use.
- **CI evaluates the exact source SHA before deployment and deploys only on gate pass** (rev 4).
- **Apex Backoffice evaluates the deployed runtime revision on demand** and shows run history.

**Non-Goals (this phase)**
- Serving production *traffic* / online evals / production monitoring (the eval deployments are real deployments, but nothing routes users to them yet).
- Gating deploys on eval results (future phase — the gate mechanics are designed to support it).

---

## 3. Repo Layout (source of truth) — agents-cli scaffold

Generated by `agents-cli create <agent> --deployment-target ...` (or `create --prototype` + `scaffold enhance` later), not hand-rolled:

```
repo/
  agents/
    <agent_name>/                      # one agents-cli project per agent
      app/
        __init__.py
        agent.py                       # model= set EXPLICITLY (see §11)
        app_utils/                     # telemetry/utility code (scaffolded)
      tests/
        eval/
          datasets/
            fast-smoke.json            # tier 1: deterministic metrics
            fast-regression.json
            judged-quality.json        # tier 2: LLM-as-judge metrics
            judged-multiturn.json
          eval_config.yaml             # metrics config (per-tier variants: eval_config.fast.yaml / eval_config.judged.yaml)
        integration/
        unit/
      pyproject.toml
  pipelines/
    azure-pipelines.yml
```

Notes:
- **Multiple datasets per agent is native**: each JSON in `datasets/` is independent; `eval generate`/`grade` operate over them. New dataset = new file.
- **Dataset authoring**: hand-write, record via `agents-cli playground` / ADK web, or bootstrap with `agents-cli eval dataset synthesize` (generates candidate cases to curate). Existing ADK `.evalset.json` files migrate via the official eval-dataset migration guide.
- **Tiering ("if warranted" judge policy)** moves from folders to **config + naming convention**: `fast-*` datasets graded with the fast metrics config, `judged-*` with the judge config. Encoded once in the pipeline/runner glue; reviewed in PRs like everything else.
- Non-prototype scaffolds include Terraform and CI/CD templates — use them as the reference for the Azure DevOps pipeline even though we author our own YAML.

## 4. The Three Surfaces

### 4.1 Local (dev inner loop)
```
agents-cli playground        # interactive dev, hot reload; record sessions into datasets
agents-cli run "..."         # one-shot terminal test
agents-cli eval generate     # run the agent over the datasets → responses
agents-cli eval grade        # score responses against eval_config metrics
```
- The generate/grade split is a feature: generate once, grade repeatedly (e.g., re-grade with a new rubric without re-running the agent).
- `agents-cli eval compare` diffs runs (regression checks between two changes); `eval analyze` inspects failures; `eval metric list` shows available metrics.
- **ALL grading locally requires GCP ADC** (`gcloud auth application-default login`) — not just the judged tier. Verified twice (build-spec E3 + re-confirmed on macOS 2026-08-15): `eval grade` hard-requires credentials at Vertex client init even when every metric is a local custom function. Day-one dev onboarding = ADC + a low-quota project. The only credential-free deterministic path is ADK-level `adk eval` fast metrics (docs explicitly state trajectory/ROUGE run locally without credentials — confirm with a 30-second run in Phase 0).

### 4.2 CI — the authoritative evaluation gate (rev 4 shape)
CI evaluates the **exact checked-out source SHA, before any deployment**, running the agent from
source inside the CI job (local `adk api_server`/`generate` — the build-spec §12-validated path).
The authoritative order:

```
checkout exact SHA → install agent + pinned toolchain → generate EVERY expected case
→ verify generated case IDs match the authored set → grade EVERY selected metric
→ verify every selected metric executed
→ gate: fail on low score | errored case | missing case | missing metric
   (infrastructure errors reported DISTINCT from quality failures)
→ publish raw + normalized results → agents-cli deploy — ONLY on pass
→ branch protection / pipeline gating makes the gate binding
```

- Per-agent path filters; pinned installs; results to GCS with full lineage.
- CI runs the **complete configured suite** — it is the authoritative quality gate; Apex runs may be
  partial (see §4.3), CI runs never are.
- Custom `custom_function` metrics execute here safely — CI already runs repo code by definition.
  The credential-bearing Runner does NOT get that privilege (§4.3).
- **Gate mechanism — RESOLVED (2026-08-15, two independent checks)**: `eval grade` never exits non-zero on low scores (verified in 1.3.1 source — only operational `ClickException`s exit 1), and Google's own scaffolded CI/CD templates contain **no eval gating at all** (PR checks run pytest unit/integration; staging runs Locust — `eval grade` appears in none of their workflows). There is no first-party mechanism to mirror: **`tools/eval_gate.py` (build-spec §6) is the gate**, not a fallback. The rev-1 pytest wrapper remains available as an escape hatch only.
- Gating posture (rev 4): `fast-` and controlled `safety-` rubric checks block immediately; `judged-`
  participates from day one and is **promoted to blocking after baseline stability evidence**
  (conservative thresholds, fixed canaries, pinned judge versions, documented rollback policy);
  `golden-` blocks when stable.
- The pipeline still never bakes agent code into a Runner image — the Runner (§4.3) is a thin
  orchestrator that evaluates *deployed* revisions; CI evaluates *source* in its own job sandbox.

### 4.3 On-demand — Apex Backoffice
Three-part design; the browser never runs evals itself. **Apex evaluates the *deployed runtime
revision*** (via `generate --url <runtime>/api`, the rev-3 spike result): it resolves the deployed
revision → resolves that deployment's source SHA → loads datasets + grading configs **from that exact
SHA (never mixed with `main`)** → runs → shows current results and history. **Security constraint
(rev 4): the credential-bearing Runner never executes arbitrary repository Python** — initially Apex
runs cloud-supported metrics only and reports repo-local custom metrics as **"not run"**, marking the
run **partial**. CI remains the authoritative complete-suite gate.

1. **UI (thin web app) — locked scope, three capabilities only:**
   - **Run evals per agent**: pick agent → pick tier/datasets (listed live from GitHub@main) → run → watch status.
   - **View agent instructions**: read-only render of the agent's definition from GitHub@main — display `app/agent.py` (or its `instruction=` block) fetched via the Contents API, labeled with the SHA. No editing; changes happen through PRs.
   - **View previous runs**: list + detail of past runs read from GCS `results/` manifests (CI and on-demand runs both, since they share the schema) — per-dataset scores, per-case pass/fail, judge rationale, token rollups, and a deep link to Cloud Trace per run.

   Explicitly **out of scope** (served elsewhere): trend dashboards and analytics (BigQuery Agent Analytics / console), trace waterfalls (Cloud Trace deep links), dataset authoring/editing (git + `agents-cli`), and interactive agent chat (playground / Agent Engine console later).
2. **Eval Runner Service (FastAPI on Cloud Run):**
   - `GET /agents` → list agents (from repo structure @ main).
   - `GET /agents/{agent}/instructions` → GitHub Contents API → `app/agent.py` @ main; return raw file + best-effort extracted `instruction` string + SHA. (Extraction is display sugar — the raw file is the source of truth; don't build logic that depends on parsing it.)
   - `GET /agents/{agent}/datasets` → GitHub Contents API (GitHub App token) → lists `agents/{agent}/tests/eval/datasets/*.json` at `main`.
   - `POST /agents/{agent}/runs` → resolves `main` to a SHA, downloads eval sets + config at that SHA, enqueues an async run (Cloud Tasks or background worker), returns `run_id` immediately.
   - Worker executes `agents-cli eval generate` + `eval grade` per dataset (or the underlying ADK APIs if in-process control proves cleaner), writes per-case results to the results store as they complete.
   - `GET /runs/{run_id}` → status + results (UI polls or uses SSE).
3. **Results store:** GCS-first (see §5). The UI reads run documents from GCS via the Runner Service; keyed by `agent / evalset / run_id / git_sha / trigger`.
4. **Judged tier on demand:** the UI exposes "Run fast", "Run judged", "Run all" — the tiers map directly to the `fast-*` / `judged-*` dataset filename prefixes (plus their per-tier `eval_config.*.yaml`) pulled from GitHub, so what runs is always what's on `main`.

**Critical coupling decision — REVISED in rev 3:** the agent under test is the **deployed Agent Platform revision**, not code baked into a Runner image. The SHA coupling becomes: *deploy step records (agent, revision/resource-name, git SHA) in the run manifest; every eval run targets a named revision and stores its SHA*. The Runner Service shrinks to a thin orchestrator (pull datasets @ SHA from GitHub → invoke `eval submit`/`generate --url` against the recorded revision → gate → write manifests to GCS). It has no per-agent dependencies, so one small image serves all agents (former G2 dissolved) and there is no image↔main drift window (former G3 dissolved — an on-demand run just targets the latest recorded deployment, labeled with its SHA).
- The rev-2 design (agent code baked into a per-SHA Runner image, `adk api_server` in-process) remains documented in build-spec §12 as the fallback if the Agent Platform eval spike fails badly.

LLM-judged metrics via the **agents-cli path** always need GCP creds (`GOOGLE_CLOUD_PROJECT` + ADC — E3). At the **ADK level** the story is softer (verified against adk.dev 2026-08-15): several judge criteria accept a plain `GOOGLE_API_KEY`; only `safety_v1` and the `multi_turn_*` family hard-require the Vertex/Agent Platform Eval SDK with a GCP project. ADK fast metrics (`tool_trajectory_avg_score`, ROUGE-based `response_match_score`) are local, credential-free, and the recommended CI defaults.

---

## 5. Results Data Model (draft)

```
EvalRun {
  run_id, agent, git_sha, trigger: ci|on_demand|local(optional),
  started_at, finished_at, status: running|passed|failed|error,
  requested_by, criteria_snapshot
}
EvalSetResult {
  run_id, evalset_id, status, num_cases, num_passed
}
EvalCaseResult {
  run_id, evalset_id, eval_case_id,
  metrics: { tool_trajectory_avg_score, response_match_score, ... },
  pass: bool,
  invocations: [                        # full observability payload per turn
    {
      prompt,                           # user_content
      expected_response, actual_response,
      expected_tool_calls: [{name, args}],
      actual_tool_calls:   [{name, args, result_summary}],
      tokens: { prompt, candidates, total },   # via capture layer (§5a)
      latency_ms, model
    }
  ],
  tokens_total: { prompt, candidates, total }, # rolled up per case
  judge_rationale, trace_ref
}
```

### 5a. Observability capture layer (tokens / tool calls / responses) — all Google-native, mostly provisioned

- **Responses + tool calls come free from the eval engine**: ADK eval results carry actual-vs-expected invocations (final response + tool trajectory per turn). With `--eval_storage_uri gs://...`, ADK persists eval results itself; we add only a thin **manifest/rollup writer** (run index, token totals, judge rationale pointers) so the UI has one small JSON to read per run.
- **Tokens/traces via built-in tooling — with one custom seam (verified 2026-08-15)**: agents-cli deploys have **Cloud Trace enabled by default**; for our CI/Runner processes, ADK ≥1.17's native OTel export (`--otel_to_cloud` / programmatic config) sends the same spans. Spans (`invoke_agent`, `call_llm`, `execute_tool`) follow OTel GenAI semantic conventions and carry prompts, responses, and token usage (`gen_ai.usage.*` attributes); Cloud Trace renders the timeline. **However**: the generate trace artifact provably cannot carry tokens (its `Event` schema has only event_id/content/timestamp/author — source-inspected), and the spans are emitted by the *agent server* process, which doesn't know our `run_id`. Correlating run→trace (e.g. an OTel resource attribute set at server start by the Runner, which controls the server lifecycle) is the one piece we must design — without it there are no per-run token rollups, `trace_ref`s, or Trace deep links.
- **Two Google-native stores, joined by IDs**: eval scores/artifacts in GCS, traces (incl. tokens) in Cloud Trace, correlated by run_id/trace_id in the manifest. The UI deep-links into Cloud Trace rather than us rebuilding a trace viewer.
- **Content logs (prompts/responses) at full fidelity**: provided by the agents-cli-provisioned GCS bucket + BigQuery dataset (`agents-cli infra`), queryable via BigQuery Agent Analytics — this is the Google-native answer to "have that data available to show."
- Token rollups per run: from trace/analytics data at manifest-write time, so cost-per-run displays without parsing traces.
**Storage: GCS as the system of record.** All three surfaces write to one bucket:

```
gs://<org>-agent-evals/
  results/                              # structured JSON written by our wrappers
    <agent>/<git_sha>/<run_id>/
      run.json                          # EvalRun
      traces/                           # generate output (populated traces JSON)
      grade_results/                    # grade output (results_<ts>.json + .html)
      gate_summary.json                 # eval_gate.py machine-readable verdict
```

- **Correction (2026-08-15):** `--eval_storage_uri` is an **ADK-level flag** (`adk eval` / `adk web` / `adk api_server`) — the agents-cli `generate`/`grade` runner writes plain local dirs (`--output`/`--traces`) and has no GCS flag. So there is no "free" GCS persistence on the rev-2 path; the mechanism is `gsutil rsync` of the artifacts dir (as the CI/Runner flows already specify). A `raw/` prefix populated by `--eval_storage_uri` applies only if the ADK escape hatch is used.
- The `results/` prefix is our normalized layer; CI and the Runner Service both write it, so one schema serves the UI and analytics.
- **BigQuery: deferred (not needed for current scope).** Nothing in the locked UI scope (run / instructions / run history) requires it — GCS manifests serve all reads. When cross-run analytics is eventually wanted, adopt the agents-cli-provisioned route (`agents-cli infra single-project` creates a BigQuery dataset for content logs / Agent Analytics via Terraform) rather than building custom. Until then, if `infra` provisioning is used for the bucket/SA, the BQ dataset can sit unused or be trimmed from the Terraform.
- Judge outputs should persist the judge's **rationale/explanation**, not just the score — that's most of the debugging value.

---

## 6. Auth Design

| Hop | Mechanism |
|---|---|
| Dev laptop → GCP (**all** `eval grade` runs — see §4.1 — plus GCS artifacts) | ADC via `gcloud auth application-default login`; devs get a low-quota project or per-team quota. Mandatory day-one onboarding, not judged-tier-only |
| Azure DevOps → GitHub (trigger + checkout) | GitHub App service connection (preferred over PAT) |
| CI job → GCP (LLM judge metrics, GCS) | Preferred: Workload Identity Federation (keyless) — **officially documented by Google end-to-end for Azure DevOps** (IAM doc "Workload identity federation with deployment pipelines": ARM WIF service connection OIDC token → workload identity pool → SA impersonation; verified 2026-08-15). Fallback: SA key in ADO secure files (works; Google discourages keys). |
| Runner Service → GCP | Cloud Run service identity (native, keyless) |
| Runner Service → GitHub | GitHub App installation token (read-only contents scope) |
| UI → Runner Service | Your standard internal SSO/IAP |

---

## 7. Rollout Plan

**Superseded — `build-spec.md` §10 is the authoritative build order and acceptance criteria.** (This section previously carried rev-1 text — pytest wrapper as primary gate, Firestore/BigQuery results store — contradicting §5's GCS-first decision and the verified rev-2 toolchain. Summary of the current plan: Phase 0 scaffold + tier configs + gate script; Phase 1 live-credential spikes; Phase 2 ADO pipeline; Phase 3 Runner Service; Phase 4 UI. Future: deploy gating, scheduled regressions, `adk conformance` as a complementary layer.)

---

## 8. LLM-as-Judge Design (all three surfaces)

**Available judge criteria (ADK built-ins, all backed by the Vertex AI Gen AI Evaluation Service):** `final_response_match_v2` (semantic match to reference), `rubric_based_final_response_quality_v1` and `rubric_based_tool_use_quality_v1` (custom rubrics, no reference needed), `hallucinations_v1` (groundedness vs tool outputs), `safety_v1`, and the `multi_turn_*` family. Deterministic tier stays on `tool_trajectory_avg_score` + `response_match_score`.

Config shape depends on the layer (they are **not** interchangeable):
- **agents-cli (our runner): `tests/eval/eval_config.judged.yaml`** — verified format (build-spec E8): `metrics_to_run:` + `custom_metrics:` (each with `custom_function_file:` or inline `custom_function:`), plus our own `thresholds:` block consumed only by `eval_gate.py` (unknown keys verified tolerated — the models are `extra="allow"`).
- **ADK escape hatch (`adk eval`): `test_config.json`** with a `criteria` dict, e.g. `{"criteria": {"final_response_match_v2": 0.75, "hallucinations_v1": 0.8}}` (defaults: trajectory 1.0 / response_match 0.8).

**Design rules:**
1. **Same engine, same config, every surface.** No surface passes ad-hoc judge settings; the tier's `eval_config.*.yaml` in the repo is the only source of judge criteria. This is what makes local/CI/UI scores comparable.
2. **Judge scores are noisier than deterministic scores.** LLM judges are non-deterministic; identical runs can score slightly differently. Consequences: (a) set thresholds with margin, not at the observed mean; (b) treat a judged-tier failure as "investigate", and consider making judged criteria non-blocking in CI initially, promoting to blocking once threshold stability is proven; (c) never diff two runs' judge scores to two decimal places and call it a regression.
3. **Pin everything pinnable.** Pin `google-adk` version in all surfaces; if/where ADK exposes judge-model selection, pin the model version too, and record it in `EvalRun.criteria_snapshot`. A silent judge-model upgrade shifts every score and poisons trends.
4. **Cost & quota controls:** judged tier is the expensive tier. Controls: per-agent run lock in the Runner Service; concurrency cap on judge-metric evals; budget alert on the Vertex eval project; CI runs judged tier only on merge to main (never per-commit); on-demand judged runs are logged with `requested_by`.
5. **Rationale is a first-class output.** Persist judge explanations per case to GCS (§5) and show them in the UI — a score without the "why" doesn't help anyone fix an agent.

---

## 9. Claim Validation Log

Every load-bearing claim was checked against primary sources (Google ADK official docs, Microsoft Azure DevOps docs) on 2026-08-15.

| # | Claim | Verdict | Source |
|---|---|---|---|
| 1 | `adk eval <agent> <evalset...> --config_file_path=... --print_detailed_results` is the CLI contract; multiple eval set files per invocation; `file.json:eval_1,eval_2` runs selected cases | ✅ Confirmed | google/adk-docs `docs/evaluate/index.md` |
| 2 | Eval sets are Pydantic-schema JSON files; an eval set holds many cases incl. multi-turn sessions; multiple sets per agent is the intended model (test files = unit-style, evalsets = integration-style) | ✅ Confirmed | same |
| 3 | `test_config.json` holds `criteria`; defaults are trajectory 1.0 / response_match 0.8 | ✅ Confirmed | same |
| 4 | `AgentEvaluator.evaluate(agent_module=..., eval_dataset_file_path_or_dir=...)` via pytest is the documented CI/CD integration path | ✅ Confirmed | same ("This approach allows you to integrate agent evaluations into your CI/CD pipelines") |
| 5 | LLM-judged criteria (response quality, safety, multi-turn) require Vertex AI Gen AI Evaluation Service + GCP auth; trajectory/ROUGE metrics recommended for CI | ⚠️ **Partially revised (2026-08-15)**: all named criteria exist, but at the ADK layer several judge criteria accept a plain `GOOGLE_API_KEY`; only `safety_v1` + `multi_turn_*` hard-require the Vertex/Agent Platform Eval SDK with a GCP project. (agents-cli `grade` hard-requires ADC regardless — build-spec E3.) Trajectory/ROUGE local + credential-free confirmed. | adk.dev/evaluate/criteria |
| 6 | Eval sets can be authored/edited/re-run in `adk web` UI incl. trace inspection | ✅ Confirmed | same |
| 7 | `adk eval` CLI exits non-zero when criteria fail | ⚠️ **Not documented — downgraded.** Docs position the CLI for automation but state no exit-code contract; a past CLI bug report shows regressions happen. **Design change:** pytest assertion failure is the CI gate; CLI reserved for local use. Verify exit behavior empirically in Phase 1 if we ever want CLI-in-CI. | adk-docs; google/adk-samples issue #96 |
| 8 | Azure DevOps pipelines can build GitHub-hosted repos via GitHub App/service connection, with CI triggers on `main` and per-path filters | ✅ Confirmed | Microsoft Learn — Build GitHub repositories; Triggers in Azure Pipelines |
| 9 | Path filters: wildcards supported, case-sensitive, no variables in trigger blocks; YAML must exist on target branch | ✅ Confirmed (gotchas added to design) | Microsoft Learn / MS Q&A |
| 10 | `PublishTestResults@2` + pytest `--junitxml` renders per-test results in ADO Tests tab | ✅ Confirmed (standard documented ADO capability) | Microsoft Learn |
| 11 | Azure→GCP Workload Identity Federation for keyless CI auth | ✅ **RESOLVED (2026-08-15): officially documented.** Google's IAM doc covers the exact ADO flow: ARM WIF service connection token → workload identity pool (attribute mapping `assertion.sub.split('/sc/')[1]`, condition on `assertion.oid`) → SA impersonation via credential-config file. Remains a config task, not a research spike. Fallback (SA key in secure files) still valid. | GCP IAM: workload-identity-federation-with-deployment-pipelines |
| 12 | ADK supports `--eval_storage_uri gs://<bucket>` for storing eval artifacts | ✅ Confirmed | ADK CLI reference |
| 13 | `adk conformance test` exists as a CLI regression tool explicitly recommended for CI/CD (replay of recorded baselines) | ✅ Confirmed — noted as future complementary layer, not core | adk-docs |
| 14 | Eval results natively contain actual vs expected invocations: final responses + tool calls (name/args) per turn | ✅ Confirmed | adk-docs custom-metrics API (`actual_invocations: list[Invocation]`); `--print_detailed_results` shows expected/actual responses and tool calls |
| 15 | Token counts are per-event `usage_metadata` (prompt/candidates/total), NOT included in eval results or default logs; must be captured explicitly | ✅ Confirmed (gap identified → §5a capture layer) | google/adk-python discussions #97, #3273, #2434 |
| 16 | ADK OTel/OpenInference instrumentation attaches prompts, responses, token usage to LLM spans and tool calls to tool spans automatically | ✅ Confirmed | Arize ADK tracing docs |
| 17 | ADK ≥1.17 has built-in OTel export to Google Cloud Observability via `--otel_to_cloud` (and `--trace_to_cloud` on deploy); spans incl. call_llm/execute_tool with GenAI semconv; Cloud Trace waterfall | ✅ Confirmed — **supersedes the custom span exporter (row 16 approach deleted)** | Google Cloud Observability docs; adk-docs Cloud Trace integration |
| 18 | ADK latest = 2.6.3 (2026-08-07), ~bi-weekly cadence; 2.0 had breaking changes (agent API, event model, session schema); 2.2.0 changed default LlmAgent model | ⚠️ **Superseded (2026-08-15): 2.7.0 released 2026-08-13** with eval-semantics changes (threshold grading, crash-fails-eval, CSV export); cadence ~weekly; 1.x line at 1.38.0. 2.0-breaking-changes and 2.2.0-default-model claims ✅ confirmed against release notes. See §11 for the pin decision. | PyPI google-adk; adk-python releases (live-checked) |
| 19 | Agent Engine: Sessions & Memory Bank GA; observability dashboard (tokens, latency, errors, tool calls), traces tab, playground, Evaluation Layer w/ User Simulator | ✅ Confirmed | Vertex AI release notes; Google Cloud blog (Agent Builder expansion) |
| 20 | agents-cli (google/agents-cli) is Google's unified ADK lifecycle CLI: `create` scaffolds agent + tests/eval/datasets + eval_config.yaml (+ Terraform/CI-CD non-prototype); `eval generate`/`grade`; also `eval dataset synthesize`, `eval compare`, `eval analyze`, `eval metric list`, `eval optimize` | ✅ Confirmed | google.github.io/agents-cli tutorial + CLI reference |
| 21 | agents-cli deploys enable Cloud Trace by default; `agents-cli infra single-project` Terraform-provisions SA + GCS bucket + BigQuery dataset for content logs & BigQuery Agent Analytics | ✅ Confirmed | agents-cli tutorial (observability section) |
| 22 | `eval grade` exit-code/threshold gating semantics for CI | ✅ **RESOLVED EMPIRICALLY (2026-08-15, v1.3.1)**: grade exits non-zero only on operational errors, never on low scores; no threshold flag. CI gate = our `tools/eval_gate.py` over `results_<ts>.json`. Full findings in `build-spec.md` §2. | Installed package source inspection + execution |
| 23 | agents-cli dataset format (`EvaluationDataset`, eval_cases w/ prompt) differs from ADK `.evalset.json`; official migration guide exists | ✅ Confirmed | agents-cli reference: "Migrating Eval Datasets" |

**Caveat:** ADK is Python-only for evaluation and evolves fast; pin the `google-adk` version in CI and re-check docs at implementation time.

---

## 11. Version Strategy (checked 2026-08-15 — re-verify at implementation)

- **ADK Python: pin 2.6.3, but note 2.7.0 shipped 2026-08-13** (yes — the cadence is ~weekly or faster, not bi-weekly; 1.x maintenance line is at 1.38.0). **2.7.0 changes eval semantics at the ADK layer**: judge/ROUGE metrics graded against criterion thresholds, agent-crash-before-metrics now fails the eval, plus eval-results CSV export and result persistence in `AgentEvaluator`. Those are exactly the behaviors we want, so **re-evaluate the pin (2.6.3 vs 2.7.0) at Phase 0** — but decide once and freeze; the agents-cli `grade` path is unaffected either way (it scores via the Vertex SDK, not ADK eval). **2.0 broke the agent API, event model, and session schema vs 1.x** — never float. (Docs domain note: google.github.io/adk-docs now redirects to **adk.dev**.)
- **Pin the agent model explicitly.** ADK 2.2.0 silently changed `LlmAgent`'s default model (gemini-2.5-flash → gemini-3-flash-preview; confirmed in the 2.2.0 release notes) ahead of the 2.5-flash retirement — **2026-10-16 per Vertex AI release notes** (note a split-brain: the Gemini API deprecations page still says "no shutdown date announced" for the same model ID). An implicit model + an ADK bump = every eval score shifts and trends are poisoned. Every agent sets `model=` explicitly; treat model bumps as reviewed changes with an expected eval-baseline reset. Current landscape (ai.google.dev models page, 2026-08-15): **`gemini-3.7-flash` is the recommended stable Flash**; `gemini-3.6-flash` (the current scaffold default) is GA but labeled previous-generation.
- Eval features are still evolving fast in 2.x (e.g., `include_intermediate_responses_in_final` for `final_response_match_v2`; local eval-run history in the CLI; user-simulation evals). Review the ADK changelog at each pinned-version bump — features keep landing that may delete our glue code.
- `--otel_to_cloud` requires ADK ≥1.17 — satisfied by any 2.x pin.
- **agents-cli**: newer project (google/agents-cli) — pin its version too, and expect faster churn than ADK. Its dataset format (`EvaluationDataset`) differs from ADK `.evalset.json`; author new datasets in agents-cli format and use the official migration guide for any existing ADK sets. Where agents-cli lacks a feature, the ADK-level tools underneath remain the documented escape hatch.

---

## 12. Vertex AI Agent Platform (Agent Engine) — ADOPTED DAY ONE (rev 3)

**Rev-3 status change: this is no longer a future phase.** The owner decision (2026-08-16) is to deploy agents to Agent Platform from day one and run evals against its APIs. The sections below were written when this was deferred; they remain accurate as a description of what the platform provides — read "would absorb" as "absorbs".

**What it is:** managed serverless runtime for agents (deploy via `adk deploy agent_engine --trace_to_cloud`), with managed Sessions and Memory Bank (GA; billed since 2026-01-28), IAM-native agent identities, and Model Armor.

**What it would absorb from this design (Nov-2025 observability/eval expansion):**
- **Observability dashboard**: token consumption, latency, error rates, and tool calls over time, per agent — a large chunk of our custom UI's read views.
- **Traces tab + playground** in the Cloud console: trace flyouts of agent action sequences, and interactive replay against past sessions — our "debug a failed eval" workflow, managed.
- **Evaluation Layer with User Simulator**: managed multi-turn simulated-user evals; complements (doesn't replace) our GitHub-versioned golden eval sets.

**What it does NOT absorb:** GitHub as eval-set source of truth, the merge-to-main CI gate in Azure DevOps, and the "run evals for a SHA on demand" trigger. Those remain ours in every scenario.

**Rev-3 adoption (was "decision framing for the deployment phase" — now decided):** the Runner Service shrinks to a thin trigger/orchestrator (pull sets from GitHub → invoke evals against the deployed revision → write manifest); console dashboards serve most observability views; the custom UI becomes a portal over run history + deep links. Costs to model day one: runtime vCPU-hours (~$0.0864/vCPU-hr, verified Dec-2025 pricing) + sessions/memory billing ($0.25/1k events, billed since 2026-01-28) — set the budget alert before the first CI-triggered deploy. The caution stands and is now load-bearing: **the managed runtime is mature; the managed eval surface (`eval submit`/`results`, Evaluation Layer) is newer and experimental in agents-cli 1.3.1** — our GitHub-versioned eval sets + gate script remain the regression backbone, whichever execution path wins the blocking spike.

---

## 14. Command Quick Reference (per flow)

| Flow | Step | Command / call |
|---|---|---|
| Local | Interactive dev | `agents-cli playground` (hot reload) / `agents-cli run "prompt"` |
| Local | Run agent over datasets | `agents-cli eval generate` |
| Local | Score responses | `agents-cli eval grade` |
| Local | Inspect / regress-diff | `agents-cli eval analyze` / `agents-cli eval compare` |
| Local | Judge-tier auth | `gcloud auth application-default login` |
| Local | Bootstrap new dataset | `agents-cli eval dataset synthesize` (then curate + commit) |
| CI | Install (pinned) | `pip install google-agents-cli==1.3.1 google-adk==2.6.3` (✅ verified names/versions) |
| CI | Deploy merged agent (rev 3) | `agents-cli deploy --project repp-501700 --region …` (target `agent_runtime` from manifest; `--no-wait`/`--status` available) |
| CI | Eval per tier (rev 3 — pending blocking spike) | Path A: `agents-cli eval submit --resource-name <revision> --dataset … --dest gs://…` → `eval results` · Path B: `agents-cli eval generate --url <endpoint>` → `eval grade` |
| CI | Gate | `python tools/eval_gate.py <results dir> --junit gate.xml` (grade/submit themselves never gate — verified) |
| CI | Publish | `PublishTestResults@2`; `gsutil cp -r artifacts/grade_results gs://<bucket>/results/<agent>/<sha>/<run>/` |
| UI/on-demand | Trigger | `POST /agents/{agent}/runs {tier}` (Runner Service) |
| UI/on-demand | Source of truth | GitHub Contents API `GET .../contents/agents/{agent}/tests/eval/datasets?ref=main` |
| UI/on-demand | Execute | worker runs `agents-cli eval generate` + `eval grade` |
| UI/on-demand | Results | manifest + grade_results → GCS; `GET /runs/{run_id}` to read |
| One-time infra | Bucket/SA provisioning | `agents-cli infra single-project --project <id>` (Terraform; BQ portion deferred) |

See diagram `04-flows-with-commands.mermaid` for the visual version of all three flows.

---

## 15. Open Decisions

0. ~~(rev 3, blocking) Eval execution path against deployed agents~~ ✅ **DECIDED 2026-08-16: `generate --url <runtime>/api`** (spike result — see revision notes). Still open from this item: staging-instance shape (one standing scale-to-zero eval deployment per agent vs deploy-per-run — cost vs isolation; the spike used scale-to-zero `--min-instances 0`, which looks right for eval instances).
1. ~~Judged tier in CI~~ **REVISED (rev 4, 2026-08-17 — supersedes the 08-16 blocking-day-one call):** judged quality **participates from day one and is promoted to blocking after baseline stability evidence** establishes an acceptable false-block rate. Requirements while promoting: conservative thresholds, fixed canary cases, pinned judge/metric versions where available, infrastructure-error separation, documented rollback/temporary-disable policy. Baseline data point: healthy scaffold run 5.0, corrupted-reference run 3.67. (`fast-` and controlled `safety-` rubric checks block immediately.)
1b. **DECIDED (owner, 2026-08-16):** CI runner — **GitHub Actions primary** (org standard is Azure DevOps, but the owner will argue for GHA's simplicity: first-party scaffold support via `--cicd-runner github_actions`, repo+CI in one place, Google-documented GitHub WIF). **All verified ADO facts in this doc are retained as the fallback port** if the org mandates ADO — the pipeline steps are identical, only the YAML dialect and auth hop change.
1c. **REVISED (owner, 2026-08-16, same day):** per-run token consumption **will be surfaced** in CI results and the UI (live during a run where possible). Mechanism — **BigQuery Agent Analytics plugin** on eval deployments (`scaffold enhance --bq-analytics`; plugin ships in pinned ADK; event rows carry `session_id`/`invocation_id`/`usage_metadata` token counts — source-verified). Attribution is clean because eval traffic runs as hardcoded user `eval-cli-user` with one session per case: per-run tokens = SUM over user+time-window (pollable live); per-case = GROUP BY session_id. **Refinement (owner, same day): CI carries zero token code** — CI is regression-gate only. The **UI backend computes tokens lazily at view time** from the manifest's time window, which works for on-demand runs (live counter while running) AND retroactively for CI runs opened in the UI. Fallback until BQ is enabled: Cloud Monitoring `token_count` time-window query (approximate; verified live with real data). Console dashboards remain the deep-link for everything else. Eval deployments: **standing scale-to-zero instances per agent**, updated by each merge deploy.
2. Whether on-demand runs may target a branch/SHA other than `main` (useful for pre-merge "run heavy evals" button) — defer.
3. Monorepo vs repo-per-agent — this doc assumes monorepo with path filters; revisit if agent count or team boundaries force a split.
4. BigQuery external tables over GCS vs native tables with a small loader — start external, revisit if query volume grows.
5. Rubric authoring ownership — rubrics/judge configs in `eval_config.judged.yaml` are product-quality definitions; decide who reviews changes (eng vs product).
6. **ADR (2026-08-16): ADK-native eval engine considered and rejected as the platform engine.** Strengths acknowledged (native thresholds in 2.7.0, richer criteria list, credential-free deterministic tier, `--eval_storage_uri`), but `adk eval` executes the agent module in-process and cannot target a deployed agent — incompatible with the rev-3 core (evals run against the deployed Agent Platform revision). Also: faster breaking-change cadence, and agents-cli is Google's forward-integrated path. Permitted hybrid edges: `adk web` for local dataset authoring (scores never persisted), `adk conformance` as the Phase-4 replay lane. **Watch item: re-evaluate if ADK eval ever gains a deployed-target (URL) mode.** Safety coverage meanwhile: owner decision — custom red-team suite as an ACTIVE `safety-` tier with a local judge on the cli engine (see eval-conventions).
