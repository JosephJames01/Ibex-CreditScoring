# Ibex Credit — setup & secrets

This repository contains **no secrets**. Everything private is created locally
or entered in the Render dashboard. This file is the checklist.

---

## 1. Files the app creates itself (gitignored — do not commit)

| File | Created when | Contains |
|---|---|---|
| `serve/users.json` | first signup | emails + PBKDF2 password hashes |
| `serve/_anchors/*.json` | each on-chain anchor | the anchored score events |
| `chain/score-event.json` | each score | the latest score event preimage |

If any of these are already tracked by git, untrack them once:

```powershell
git rm --cached serve/users.json score-event.json
git log --all -- .env serve/users.json    # should print nothing
```

## 2. Local development (PowerShell, from the repo root)

```powershell
$env:IBEX_ARTIFACTS="artifacts_v5"
$env:OBCREDIT_ARTIFACTS="artifacts_v5"
$env:IBEX_OB_MATRIX="C:\Users\Josep\Downloads\obcache_b22\ob_matrix_full_all.pkl"
$env:TRUELAYER_CLIENT_ID="sandbox-ibexcredit-132352"
$env:TRUELAYER_CLIENT_SECRET="<your sandbox client secret>"
$env:TRUELAYER_REDIRECT_URI="http://localhost:8000/callback"
$env:TRUELAYER_SANDBOX="1"
$env:IBEX_CHAIN_NETWORK="polygon"
$env:IBEX_CHAIN_DIR="C:\Users\Josep\Downloads\hackathon_2026\ibex-smart-contract-demo-v2"
$env:IBEX_SCORE_EVENT_PATH="C:\Users\Josep\Downloads\hackathon_2026\ibex-smart-contract-demo-v2\score-event.json"
py -3.13 -m uvicorn serve.app:app --port 8000
```

Note: the repo also vendors the V2 chain project at `chain/`. Locally you can
point `IBEX_CHAIN_DIR` at either the vendored copy or your standalone clone —
they are the same code. The standalone clone keeps its own `.env` with the
real `PRIVATE_KEY` and `USER_SALT`; the vendored copy must never contain one.

## 3. Chain configuration

Locally, the chain project reads `chain/.env` (or the standalone clone's):

```
PRIVATE_KEY=<issuer wallet private key>
ISSUER_ADDRESS=0x4bCa26d44634966C75abdBCec41DDf94a930a49c
SCORE_EVENT_FILE=./score-event.json
USER_SALT=<64-hex salt — never rotate mid-demo; every anchor depends on it>
POLYGON_RPC_URL=https://polygon-bor-rpc.publicnode.com
SCORE_AUDIT_V2_CONTRACT_ADDRESS=0x8621D09F08C2f58803e7239F8D46D444e0eF63e1
POLYGON_EXPLORER_BASE_URL=https://polygonscan.com
```

## 4. Render (Docker — the fully working deployment)

The repo root `Dockerfile` builds Python 3.12 + Node 20 + the vendored chain
project. Render auto-detects it; `render.yaml` sets `runtime: docker`.

Set these in the service's **Environment** tab:

| Variable | Value | Notes |
|---|---|---|
| `TRUELAYER_CLIENT_ID` | `sandbox-ibexcredit-132352` | |
| `TRUELAYER_CLIENT_SECRET` | the real sandbox secret | secret |
| `TRUELAYER_REDIRECT_URI` | `https://<service>.onrender.com/callback` | must also be registered in the TrueLayer console |
| `PRIVATE_KEY` | the issuer wallet key | secret; the chain scripts inherit it — dotenv does not override existing env vars |
| `USER_SALT` | the same 64-hex salt as every anchor so far | secret; do not rotate |
| `SCORE_AUDIT_V2_CONTRACT_ADDRESS` | `0x8621D09F08C2f58803e7239F8D46D444e0eF63e1` | declared in render.yaml |
| `POLYGON_RPC_URL` | `https://polygon-bor-rpc.publicnode.com` | declared in render.yaml |
| `TRUELAYER_SANDBOX` | `1` | declared in render.yaml |
| `PYTHON_VERSION` | `3.12.7` | declared in render.yaml |

Baked into the image (Dockerfile `ENV`), no dashboard entry needed:
`IBEX_ARTIFACTS=artifacts_v5`, `OBCREDIT_ARTIFACTS=artifacts_v5`,
`IBEX_CHAIN_DIR=/app/chain`, `IBEX_CHAIN_NETWORK=polygon`,
`SCORE_EVENT_FILE=./score-event.json`.

Leave unset on Render: `IBEX_OB_MATRIX` (its absence is what makes the admin
cohort fall back to the bundled 100-row sample).

## 5. The admin cohort on Render (512 MB)

The full OB matrix is gigabytes and is **not** in this repo. The admin cohort
falls back to `fixtures/ob_matrix_admin100.pkl` — 100 rows sampled from the
evaluation slice, small enough to commit — whenever the full matrix is absent.

Regenerate it (once, on the research machine, with `IBEX_OB_MATRIX` set):

```powershell
py -3.13 scripts\make_admin_sample.py
git add fixtures/ob_matrix_admin100.pkl
```

## 6. Known deployment limitation

Render's filesystem is ephemeral: redeploys wipe `serve/users.json` and
`serve/_anchors/`. Accounts re-register in seconds and the chain records are
untouched; the business page's file-based verification keeps working
regardless. The production fix is a persistent disk or managed database.
