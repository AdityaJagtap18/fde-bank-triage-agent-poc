# Bank Transaction-Triage Agent — Container Apps Deploy, CI/CD & LLMOps

> A Forward Deployed Engineering (FDE) proof of concept. This repository contains both the **pre-transformation** (before) and **post-transformation** (after) states of the solution, so the impact of the work can be seen side by side.

---

## 1. Overview

| Item | Details |
|------|---------|
| **POC Name** | Bank Transaction-Triage Agent — Production Deploy, CI/CD & Rollback |
| **Scenario / Customer** | Illustrative: a bank's transaction-triage agent that must go live with an eval gate and a rollback runbook |
| **Owner** | adiraj2503@gmail.com |
| **Status** | Completed |
| **Program** | FDE Program, Assignment 7 (capstone) |

**One-line summary:** Shows that an AI agent can be containerised, deployed to Azure Container Apps with blue/green releases, and protected by an automated eval gate and a rehearsed SEV-1 rollback, so a bad policy or prompt change never reaches users.

---

## 2. Problem Statement

A transaction-triage agent decides whether each payment is **approved**, sent to **human review**, or **blocked**. Its behaviour is driven by a versioned policy (the "prompt"), so a single bad edit can change decisions in production without any code change.

- A regressed policy can look fine but silently approve high-value transactions that should be blocked.
- Without an automated gate, nothing stops a bad policy change from being promoted to production.
- When an incident happens at 02:00, there is no rehearsed, fast path back to the last good version.
- Releases that replace the live version in place have no safe way to test before taking traffic.

---

## 3. Pre-Transformation (Before)

The starting state: the agent and its deployment pipeline as first built, with the automation wired up in CI but the repo still tied to a template-style setup (blank-slate config, CI workflows, trainer-style publishing steps).

| Aspect | Current State |
|--------|---------------|
| Process | FastAPI service deployed via shell scripts (`deploy/01`–`04`) and two GitHub Actions workflows (`deploy.yml`, `sev1_drill.yml`) |
| Model / Tooling | Rule-based triage policy (`policy_v1.json`); Docker, Azure Container Registry, Azure Container Apps, GitHub Actions |
| Cost | Not measured in this POC |
| Latency / Performance | Not measured in this POC (a Locust load test is provided to measure it) |
| Accuracy / Quality | Golden-set eval gate with a 95% pass threshold |
| Pain points | Config ships with pre-filled demo values (`RG`, `ACR`, etc.); scripts call `python`, which fails on systems where only `python3` exists; CI-oriented files mixed into the app folder |

**Location in repo:** [Pre-Transformation/](Pre-Transformation/)

---

## 4. Post-Transformation (After)

The end state: the same service with cleaner, more portable deployment configuration.

| Aspect | New State |
|--------|-----------|
| Process | Same blue/green flow: deploy GREEN at 0% traffic, smoke test, eval gate, then promote or abort |
| Model / Tooling | Same stack; `deploy/03_bluegreen_promote.sh` now calls `python3` so it runs on Linux and cloud shells without a `python` alias |
| Cost | Not measured in this POC |
| Latency / Performance | Not measured in this POC |
| Accuracy / Quality | Unchanged: golden set passes on `policy_v1`, fails on `policy_v2_bad` |
| Benefits | `.env.example` now marks `RG` and `ACR` as **required** and leaves them blank, so a deployment cannot silently target someone else's resources; optional overrides are commented out |

**Location in repo:** [Post-Transformation/](Post-Transformation/)

---

## 5. Before vs. After

The two states are functionally the same agent. The differences are in configuration and portability, summarised from a file-by-file comparison of the two folders:

| Item | Pre-Transformation | Post-Transformation | Change |
|------|--------------------|---------------------|--------|
| `.env.example` | Pre-filled demo values for every setting | `RG` and `ACR` blank and marked required; the rest optional | Safer defaults |
| `deploy/03_bluegreen_promote.sh` | Calls `python` | Calls `python3` | More portable |
| `.github/workflows/` | `deploy.yml`, `sev1_drill.yml` present | Not included in this folder | Pipeline files kept in Pre only |
| `.gitignore`, `.dockerignore` | Present | Not included in this folder | Kept in Pre only |
| App code, policies, eval gate, golden set, load test | Identical | Identical | None |

> No latency, cost or accuracy deltas are reported because none were measured. The eval gate and Locust load test in this repo are the tools to capture them.

---

## 6. Approach

1. **Discover** – Understand the use case: a bank triage agent whose decisions depend on a versioned policy, where a bad policy change is the main production risk.
2. **Baseline** – Build the service (`/health`, `/ready`, `/decide`, `/version`, `/modelcard`) and a golden set of expected decisions.
3. **Evaluate** – Compare the good policy (`policy_v1`) against a deliberately regressed one (`policy_v2_bad`) using the eval gate.
4. **Build** – Containerise (non-root image with health check), push to ACR, deploy to Container Apps with probes and HTTP autoscaling (1 to 10 replicas, 50 concurrent requests each), and add the GitHub Actions blue/green pipeline.
5. **Measure** – Run the eval gate, smoke test and Locust load test; generate a model card per release.
6. **Hand off** – Provide a setup guide, an outcomes guide and a SEV-1 rollback runbook (in each folder) plus this README.

### Decision Criteria

| Criterion | Weight | Why |
|---|---|---|
| Safety of release | High | A bad policy must be blocked before it takes production traffic |
| Speed of recovery | High | Rollback should be a traffic shift measured in seconds, not a redeploy |
| Reproducibility | Medium | Same steps must work locally, in CI and on Azure |
| Simplicity | Medium | Small runtime dependencies and plain shell scripts keep the POC easy to hand over |

---

## 7. Repository Structure

```
Assignmnet7/
├── README.md
├── Pre-Transformation/              # "Before" state
│   ├── app/                         # FastAPI service, triage logic, policies, model card
│   ├── deploy/                      # 00_env, 01_acr_build_push, 02_deploy, 03_bluegreen_promote, 04_rollback
│   ├── eval/                        # eval_gate.py + golden_set.json
│   ├── load/                        # locustfile.py (autoscaling load test)
│   ├── scripts/                     # smoke_test.py, model_card.py
│   ├── diagram/                     # CI/CD blue/green diagram (png, svg)
│   ├── .github/workflows/           # deploy.yml, sev1_drill.yml
│   ├── Dockerfile, .env.example, requirements-dev.txt
│   └── setup guide (.docx), outcomes guide (.docx), rollback runbook (.xlsx)
└── Post-Transformation/             # "After" state (same layout, updated config and scripts)
```

---

## 8. Getting Started

### Prerequisites

- Python 3.12
- For the Azure steps: an Azure subscription, an existing resource group, Azure CLI (`az`), and permission to create a Container Registry and Container Apps environment
- For CI: a GitHub repository, plus the GitHub CLI (`gh`) if you want to drive workflows from the terminal

### Setup

```powershell
cd Post-Transformation
py -3.12 -m venv .venv
.venv\Scripts\pip install -r requirements-dev.txt
copy .env.example .env      # then set RG and ACR
```

### Run

Offline (no Docker, no Azure):

```powershell
# run the agent locally, then open http://localhost:8000/health
uvicorn main:app --app-dir app --port 8000

# eval gate on the good policy -> PASS (exit 0)
python eval/eval_gate.py

# eval gate on the bad policy -> FAIL (exit 1), the deploy would be blocked
$env:AGENT_POLICY = "app/policies/policy_v2_bad.json"; python eval/eval_gate.py

# smoke test
python scripts/smoke_test.py
```

On Azure (bash or Cloud Shell, after filling in `.env`):

```bash
source deploy/00_env.sh
./deploy/01_acr_build_push.sh        # build and push the image to ACR
./deploy/02_deploy_containerapp.sh   # deploy with probes and autoscaling
./deploy/03_bluegreen_promote.sh     # deploy GREEN, run eval gate, promote
./deploy/04_rollback.sh              # SEV-1 rollback drill
```

Load test the autoscaling: `locust -f load/locustfile.py --host http://localhost:8000 --headless -u 50 -r 25 -t 30s`

More detail (GitHub publishing, OIDC login, repo variables, troubleshooting, running the SEV-1 drill from CI) is in each folder's own `README.md`.

---

## 9. Results & Learnings

- **Key result:** The eval gate passes the good policy and fails the regressed one, so a bad policy change is stopped before promotion.
- **What worked:** Blue/green with a 0% traffic GREEN revision lets the smoke test and eval gate run against the real deployment before any user is affected. Rollback is a traffic shift to the previous revision.
- **What didn't:** The gate is the only barrier. The SEV-1 drill's `deploy-bad` action deliberately skips it, which shows how exposed production is without it. Deploying bad twice before rolling back leaves no good revision to fall back to unless one is named explicitly.
- **Recommendations:** Keep the golden set growing with real incident cases, always run the gate against GREEN rather than only in-process, and rehearse the rollback runbook regularly.

> Caveats: Illustrative scenario and rule-based policy. No production traffic, cost, latency or accuracy figures were measured, so none are claimed. The Pre and Post folders differ only in configuration and script portability.

---

## 10. Next Steps

- [ ] Capture baseline latency and throughput with the Locust load test and add them to the Before vs. After table
- [ ] Expand the golden set beyond the current cases, including edge cases around the block threshold and sanctioned countries
- [ ] Add alerting on Application Insights so a bad release triggers a page, not just a failed gate
- [ ] Restore the CI workflows to the Post-Transformation folder so both states are self-contained

---

## 11. Contact

adiraj2503@gmail.com
