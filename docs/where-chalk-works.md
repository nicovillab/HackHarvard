# Where Chalk works

Chalk is the coaching brain: TypeSafe Jev (System One) classifies whether a draft should stay, switch models, or hop sites. The Model Gateway holds provider keys and budgets. Offline evaluations use `chalkcompute`.

Overview: [root README](../README.md) and [Backend architecture](backend-architecture.md). Cross-provider coach: [General coach](general-coach.md). Extension side: [Extension architecture](extension-architecture.md). Extension contracts: [coaching-integration.md](coaching-integration.md). Account setup: [chalk-setup.md](chalk-setup.md). Day-to-day commands: [chalk-runbook.md](chalk-runbook.md). CLI notes: [chalk-cli-reference.md](chalk-cli-reference.md).

## In the product

| Path | Role |
| --- | --- |
| `POST /coaching/auto-pair` or `/coaching/pair` | Mint a session; store SHA-256 of the token in SQLite. Raw bearer goes to the extension only |
| `POST /classify` | Bearer session → classifier → `ClassifyResponse` (suggest switch or abstain). Raw prompt is not persisted; HMAC digest only |
| `POST /coaching/events` | `accepted` / `dismissed` / `ignored` (and richer draft-lifecycle events from the helper) |
| `GET /coaching/insights` | Aggregate history for the popup (no prompt text) |

Code: [`backend/coaching/`](../backend/coaching/) — `router.py`, `service.py`, `repository.py`, `chalk_features.py`, policy + rubric assets.

Live inference (when configured):

- Native System One: `{CHALK_ROUTER_BASE_URL}/v1/systemone` with `Authorization` + `X-Chalk-Env-Id`
- ~700 ms application deadline (not a measured SLA)
- Profile + shadow/suggest policy gate what the UI may show
- `PROMPTPRINT_MODE=mock` and `COACHING_MODE=shadow` are the safe defaults

**Key boundary:** `CHALK_ROUTER_API_KEY` stays on the backend. The browser never gets a gateway key. Origins in `ALLOWED_ORIGINS` are CORS, not auth.

## Offline evals

[`evals/`](../evals/) prepares datasets, runs the classifier locally or on Chalk, scores parsers, and publishes reports.

```sh
uv sync --extra eval
uv run python -m scripts.chalk_doctor
uv run uvicorn backend.main:create_app --factory --host 127.0.0.1 --port 8000
uv run python -m scripts.pair_extension
```

Mock pipeline and publication steps: [chalk-runbook.md](chalk-runbook.md). Live smoke and parser contract: [try-it-with-droid.md](try-it-with-droid.md).

## Honest limits

- Mock/shadow outputs are wiring checks, not measured energy savings
- Live account calls and remote evaluation runs need env secrets and dashboard setup ([chalk-setup.md](chalk-setup.md))
- No footprint coefficients → savings fields stay null until the real model lands
