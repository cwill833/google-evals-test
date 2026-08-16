# Live Validation Runbook (run on YOUR machine)

Goal: close every remaining 🔶 in build-spec.md with one live session. ~30 min, cost ≈ pennies
(a few small Gemini calls). Paste the OUTPUTS back to Claude — never the key itself.

> **Update 2026-08-15:** several items were already closed offline on this machine — the
> `thresholds:` key is tolerated by grade, `eval submit`/`eval results` are explained, the E3
> ADC requirement and E1 exit behavior were re-confirmed, and ADO→GCP WIF is now officially
> documented by Google. What still needs LIVE credentials: Spikes A, B, C, D below and the
> Cloud Trace check.
>
> **Rev 3 (2026-08-16):** all model access is Vertex ADC on `repp-501700` — the old
> AI-Studio-key path is gone (step 0 rewritten). SPIKE D added: it is the rev-3 BLOCKING
> spike (deploy to Agent Runtime + eval against the deployed revision) and now matters more
> than A–C, which validate the local/fallback path.

## 0. One-time setup (rev 3 — Vertex-only on project `repp-501700`; NO AI Studio)
1. In the Cloud console on project `repp-501700`: attach billing + set a $10 budget alert.
2. Auth + APIs (gcloud already installed):
   ```bash
   gcloud auth application-default login
   gcloud config set project repp-501700
   gcloud services enable aiplatform.googleapis.com storage.googleapis.com cloudtrace.googleapis.com
   ```
3. The venv + scaffolded `test-agent` already exist at `validation/` in this repo.
   Edit `app/agent.py`: pin an explicit stable model — recommended `MODEL = "gemini-3.7-flash"`
   (current stable Flash per ai.google.dev). Scaffold default `gemini-3.6-flash` is GA but
   previous-generation. (NOT gemini-2.5-flash — retires 2026-10-16.)
4. `.env` (already gitignored) — use the scaffold's Vertex default block:
   ```
   GOOGLE_GENAI_USE_VERTEXAI=true
   GOOGLE_CLOUD_PROJECT=repp-501700
   GOOGLE_CLOUD_LOCATION=global
   ```

## 1. SPIKE A — live generate (closes: model call works, trace artifact shape)
```bash
agents-cli eval generate --dataset tests/eval/datasets/basic-dataset.json --output artifacts/traces/
echo "GENERATE EXIT: $?"
ls artifacts/traces/
python3 -c "import json,glob; d=json.load(open(sorted(glob.glob('artifacts/traces/*.json'))[-1])); c=d['eval_cases'][0]; print(list(c.keys())); print(json.dumps(c,indent=1)[:1200])"
```
**Paste back:** exit code, file list, and that JSON snippet (redact nothing — it's just agent chat).
**Expect:** exit 0, one `traces_*.json`, cases now containing `responses` (and tool-call data for the weather case).

## 2. SPIKE B — live grade with the scaffold judge (closes: results JSON schema → gate fixture)
```bash
agents-cli eval grade --traces artifacts/traces --config tests/eval/eval_config.yaml
echo "GRADE EXIT: $?"
ls artifacts/grade_results/
python3 -c "import json,glob; d=json.load(open(sorted(glob.glob('artifacts/grade_results/results_*.json'))[-1])); print(json.dumps(d,indent=1)[:1500])"
```
(Runs on the ADC you set in step 0 — the API-key question is moot under rev 3.)
**Paste back:** exit code + the results JSON snippet (this becomes the gate script's parser fixture).

## 3. SPIKE C — hosted-agent on-demand flow (closes: end-to-end §12 protocol incl. model)
Terminal 1: `adk api_server . --port 8765`
Terminal 2:
```bash
agents-cli eval generate --dataset tests/eval/datasets/basic-dataset.json \
  --output artifacts/traces_hosted/ --url http://localhost:8765 --app-name app
echo "HOSTED GENERATE EXIT: $?"
```
**Paste back:** exit code + last ~10 lines of both terminals.

## 4. SPIKE D — THE REV-3 BLOCKING SPIKE: deploy to Agent Runtime + eval the deployed revision
```bash
# 1. Give the test-agent a deployment target (it was scaffolded --prototype):
agents-cli scaffold enhance         # choose agent_runtime; or re-create with -d agent_runtime
# 2. Deploy:
agents-cli deploy --project repp-501700 --region us-central1
agents-cli deploy --list            # note the resource name / endpoint it reports
# 3a. PATH A — native cloud-side eval (experimental):
gsutil mb -l us-central1 gs://repp-501700-agent-evals   # once
agents-cli eval submit --resource-name <from --list> \
  --dataset tests/eval/datasets/basic-dataset.json --dest gs://repp-501700-agent-evals
agents-cli eval results             # follow its help for the run id arg
# 3b. PATH B — only if deploy reported an HTTP endpoint:
agents-cli eval generate --dataset tests/eval/datasets/basic-dataset.json \
  --output artifacts/traces_deployed/ --url <endpoint> --app-name app
```
**Paste back:** every exit code + last ~15 lines of each command (especially any error naming the
protocol/route) + what `deploy --list` shows. This single spike decides the CI execution step
(arch §15 decision 0). Path A working = CI uses submit/results. Path B working = CI uses
generate→grade. Both failing = fallback to build-spec §12 (self-hosted api_server), rev 3 revisited.

## 5. Cloud Trace check (after any spike above)
```bash
adk api_server . --port 8765 --otel_to_cloud   # rerun spike C against it
```
Then console → Trace explorer. For the deployed agent (Spike D), traces should appear with no
flags — Cloud Trace is default-on for agents-cli deploys.
**Paste back:** whether spans appear and whether token counts show on the call_llm spans
(`gen_ai.usage.*` attributes) — this decides the G1 token-capture design.

## 6. Forced-failure check (closes: gate design assumption)
Edit `tests/eval/datasets/basic-dataset.json`: change the Paris reference to something wrong,
rerun generate+grade, and paste the scores — proving low scores still exit 0 (why eval_gate.py exists).

## What Claude does with your outputs
- Fixture the real results JSON into the gate-script spec (§6) so a coding agent can write the parser test-first.
- Close/annotate every 🔶 row in build-spec §2/§4 with your observed behavior.
- Adjust the Runner Service spec if the hosted flow surprised us anywhere.
