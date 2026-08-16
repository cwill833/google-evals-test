# Build Handoff Brief — Agent Eval Platform

**For:** the coding agent taking over this project.
**Target repo:** https://github.com/cwill833/google-evals-test.git (empty; you build the monorepo here)
**GCP project:** `repp-501700` (billing linked, $10 budget alert, APIs enabled, ADC working on the
owner's machine; a live scale-to-zero test deployment exists at
`reasoningEngines/2835184740464590848`, us-central1)
**CI:** GitHub Actions (owner decision; this personal repo doubles as the GHA argument prototype).
GitHub→GCP auth via Workload Identity Federation (`google-github-actions/auth`).

## Read in this order
1. `agent-eval-architecture.md` — the why; rev-3 revision notes at top are the current truth.
2. `build-spec.md` — the how; §2 (E-findings) and §2a (L-findings) are empirically verified facts —
   treat them as constraints, not suggestions.
3. `eval-conventions.md` — the dataset/tier/metric taxonomy you will encode in `tools/`.
4. `architecture-review.md` — audit trail; the G-items record why design decisions exist.
5. `live-validation-runbook.md` — what was validated and how; rerun pieces if behavior seems off.
6. `fixtures/` — REAL captured artifacts: `grade_results_fixture.json` (write the gate parser
   test-first against this), `traces_fixture.json` (dataset/trace shape).

## Non-negotiable verified constraints (violating these = bugs)
- Pins: `google-adk==2.6.3`, `google-agents-cli==1.3.1`, model `gemini-3.7-flash`, region
  `us-central1`. Vertex-only auth (`GOOGLE_GENAI_USE_VERTEXAI=true`); NO AI Studio keys anywhere.
- `eval grade` NEVER fails on low scores → `tools/eval_gate.py` is the only gate (build-spec §6).
- `generate` exits 0 on partial success and silently drops failed cases → the gate must compare
  graded-case-count against the dataset's case count (L2).
- Eval path against deployed agents: `generate --url <runtime>/api` — the `/api` suffix is required;
  `eval submit` is shelved (non-functional, L7).
- Eval server sessions: start any locally-hosted `adk api_server` with `--session_service_uri
  memory://` (persistent `.adk` storage is the default — L-corrected §12).
- Reference-based metrics require `reference` on every case in the dataset — enforce in
  `tools/eval_lint.py` (G6; Google's own SDK crashes on violations).
- `generate` must run from the agent's project root (walks up for `agents-cli-manifest.yaml`).

## Phases (walking skeleton — each phase ends with something demonstrably working E2E)

### Phase 0 — Repo bootstrap + the one custom component
Build: monorepo per build-spec §3 in the target repo; port the validated agent from `validation/
test-agent/` (already pinned + configured) as `agents/demo-agent/`; `docs/` moves in as repo docs;
tier configs `fast` + `judged` (+ `golden` template) per eval-conventions; `tools/eval_tiers.py`,
`tools/eval_lint.py`, `tools/eval_gate.py` — gate written TEST-FIRST against
`fixtures/grade_results_fixture.json`; GitHub Actions PR check running lint + gate unit tests
(no GCP creds needed).
**Done when:** local `generate → grade → gate` passes on demo-agent; a forced-fail dataset makes the
gate exit 1 (baselines: healthy 5.0 / corrupted 3.67); lint rejects a golden-tier fixture missing a
reference.

### Phase 1 — CI on merge (GitHub Actions → GCP)
Build: WIF setup (`google-github-actions/auth`); merge-to-main workflow: pinned installs → deploy
standing scale-to-zero eval instance (`agents-cli deploy -d agent_runtime --min-instances 0`) →
per-tier `generate --url <runtime>/api` → `grade --config <tier>` → gate (fast blocks; judged blocks
with generous thresholds) → `gsutil rsync` results to
`gs://repp-501700-agent-evals/results/<agent>/<sha>/<run_id>/` + append `index.json` + `run.json`
manifest (with `deployed_revision`, `criteria_snapshot`, timestamps).
**Done when:** a merge produces a complete run in GCS; a deliberately broken agent fails at generate
(loudly); the forced-fail dataset blocks the merge.

### Phase 2 — Walking-skeleton Runner + UI (the owner's explicit ask)
Build: minimal FastAPI Runner on Cloud Run (§8 endpoints: agents / datasets / POST runs / GET run /
GET runs — thin orchestrator, NO agent code baked in) + a deliberately simple single-page UI: agent
list → trigger run (tier picker) → live status poll → run history from `index.json` → run detail
(gate summary, per-case scores, judge rationale, token display, Agent Platform console deep link).
Tokens: lazily computed at view time (L11) — Monitoring window-query fallback until Phase 3's BQ.
**Done when:** the full loop is demonstrable: local run → CI run on merge → on-demand run from the
UI → all three visible in run history → console deep link shows the deployed agent's dashboards.

### Phase 3 — Hardening + full taxonomy
Build: enable `--bq-analytics` on eval deployments; switch token display to the BQ query (live counter
for in-flight runs; per-case GROUP BY session_id); per-agent run lease (GCS lease file honored by BOTH
CI and Runner — prevents concurrent runs against one standing instance AND deploys mid-run); remaining
tier configs (grounded / safety / multiturn) per eval-conventions; instructions screen (GitHub
Contents API); CODEOWNERS paths; `eval-metrics.snapshot` + registry-diff check.
**Done when:** two simultaneous run requests serialize cleanly; tokens show per-case; a PR adding a
new premade metric to the snapshot shows the intended diff.

### Phase 4 — Future lanes (backlog, not now)
Baseline-relative gating (`eval compare`), synthesized multiturn datasets, `eval submit` re-test on
newer agents-cli, conformance lane, `eval optimize`.

## Shared-machine etiquette (OTHER AGENTS ARE BUILDING OTHER PROJECTS ON THIS MACHINE)
- **Port block: this project owns 8600–8699. Never bind outside it.** Assignments: Runner FastAPI dev
  `8610` · UI dev server `8620` · local `adk api_server` `8630` · `agents-cli playground` `8640` ·
  ephemeral test servers `8650–8699`. If a port in the block is busy, take the next free one IN the
  block — never someone else's.
- Never kill, restart, or send signals to processes you did not start. Identify yours by PID you
  recorded at spawn, not by port-scanning or name-matching.
- Stay inside your clone of `google-evals-test` (plus the read-only reference paths below). Do not
  create, edit, or delete files elsewhere; use your own scratch dir for temp files.
- Scope every git command to your clone (run from inside it); never `git config --global`.
- Stop background servers you started when a phase's work is done.
- GCP: project `repp-501700` is shared with the validation work — reuse the existing bucket
  (`gs://repp-501700-agent-evals`) and deploy your agent under its own display name (`demo-agent`);
  don't delete or redeploy `test-agent` (`reasoningEngines/2835184740464590848`) — it's the owner's
  validation instance.

## Local machine facts (valid on the owner's machine)
- gcloud at `~/google-cloud-sdk/bin` (add to PATH in your shells); ADC + CLI auth working; project set.
- Pinned venv already built: `…/Playgrounds/google-evals/validation/evals` (read-only reference —
  make your own venv in your clone with the same pins).
- The validated agent to port lives at `…/Playgrounds/google-evals/validation/test-agent` (read-only
  reference; copy, don't move; its `.env` uses the Vertex trio — replicate, never add API keys).

## Open items the builder inherits (small, listed honestly)
- Cloud Trace span destination for runtime telemetry unconfirmed (L10) — investigate only if span
  waterfalls are wanted; tokens don't depend on it.
- BQ plugin live-streaming latency unverified (source-verified only) — Phase 3 acceptance covers it.
- Premade-metric smoke results for golden/grounded/safety tiers: see build-spec §2a (L12, added after
  the live batch run) — adopt its per-metric notes into the tier configs.
- UI auth: none for the prototype repo; IAP/SSO decision deferred to org deployment.
