# Ibex Credit — setup & secrets

This repository contains **no secrets**. Everything private is created locally
or entered in the Render dashboard. This file is the checklist.

---

## 1. Files the app creates itself (gitignored — do not commit)

| File | Created when | Contains |
|---|---|---|
| `serve/users.json` | first signup | emails + PBKDF2 password hashes |
| `serve/_anchors/*.json` | each on-chain anchor | the anchored score events |
| `score-event.json` (chain repo) | each score | the latest score event preimage |

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

Adjust `IBEX_CHAIN_DIR` / `IBEX_SCORE_EVENT_PATH` to wherever the chain repo
(`hackathon_2026`, branch `feature/ibex-smart-contract-demo`, folder
`ibex-smart-contract-demo-v2`) is cloned. Without them the score page works
but anchoring is skipped.

## 3. Chain repo secrets (`.env` inside `ibex-smart-contract-demo-v2`)

Never committed — the chain repo has its own `.gitignore` for this:

```
PRIVATE_KEY=<issuer wallet private key>
ISSUER_ADDRESS=0x4bCa26d44634966C75abdBCec41DDf94a930a49c
SCORE_EVENT_FILE=./score-event.json
USER_SALT=<64-hex salt — must match every anchor, never rotate mid-demo>
POLYGON_RPC_URL=https://polygon-bor-rpc.publicnode.com
SCORE_AUDIT_V2_CONTRACT_ADDRESS=0x8621D09F08C2f58803e7239F8D46D444e0eF63e1
POLYGON_EXPLORER_BASE_URL=https://polygonscan.com
```

## 4. Render dashboard env vars

Set in the service's Environment tab (`render.yaml` declares the rest):

- `IBEX_ARTIFACTS=artifacts_v5` and `OBCREDIT_ARTIFACTS=artifacts_v5` —
  **without these the service silently serves the old model**
- `TRUELAYER_CLIENT_ID`, `TRUELAYER_CLIENT_SECRET`, `TRUELAYER_SANDBOX=1`
- `TRUELAYER_REDIRECT_URI=https://<your-service>.onrender.com/callback`
- `IBEX_CHAIN_NETWORK=none` — Render's Python runtime has no Node, so the
  chain demo runs locally
- `PYTHON_VERSION=3.12.7`

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
