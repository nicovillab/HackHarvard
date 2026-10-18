# Where Daytona works

Ephemeral Linux sandboxes hold private chat exports and agent-written code. Only checked results leave. API keys never go into the sandbox.

Overview: [root README](../README.md) and [Backend architecture](backend-architecture.md). Droid jobs: [Where Droid works](where-droid-works.md).

## Why

Two things should never live on our server disk: users' private chat history, and code an AI agent just wrote. The sandbox holds both.

| Sandbox | In | Out | Never out |
| --- | --- | --- | --- |
| **Upload** (one per upload) | Job code, export (from memory), then Droid-written `parser.py` / `analyze.py` | Redacted copy, `insights.json`, `summary.json` | Export, `prompts.jsonl` |
| **Repair** (one per capture failure) | `clean_snapshot.py` + raw page snapshot | Cleaned page | Raw snapshot |

Code: [`run_hybrid_jobs`](../backend/sandbox.py) and [`clean_snapshot_in_sandbox`](../backend/sandbox.py), via [`DaytonaExecutor`](../backend/executor.py). Inside the upload sandbox: [`run_real.py`](../sandbox_jobs/run_real.py).

## Hybrid mode (default when `DAYTONA_API_KEY` is set)

Our tier blocks outbound traffic to Factory domains and refuses custom snapshots. So Droid itself does **not** run in Daytona. Split:

### Upload

1. **Redact in sandbox.** `/upload` reads the export into memory (50 MB cap; no `UploadFile` temp spill). Upload bytes into a fresh sandbox; run [`redact.py`](../sandbox_jobs/lib/redact.py); download **only the redacted copy**.
2. **Droid on backend.** `ingest.py` then `analyze.py` with `REDACTED_INPUT=1`. First-pass validators on redacted data.
3. **Scripts on real export in sandbox.** Upload Droid's scripts; `run_real.py` requires counts to match. Only `insights.json` and `summary.json` download. Parser cached only after this pass.
4. **Delete** in `finally`.

### Repair

1. Clean raw snapshot in a short-lived sandbox; fail closed if email or conversation links remain.
2. Delete sandbox before Droid starts.
3. Mechanic + [`check_adapter.py`](../sandbox_jobs/lib/check_adapter.py) on backend; publish only if checker, keys, and version pass.

## Privacy

Goal: Droid never sees real chat text (anything it reads leaves via the model provider).

- Ingestor: redacted copy — structure kept, text → `"<text N chars>"`. Short single-token fields (roles, model names) kept.
- Analyst: fake `prompts.jsonl` + `aggregates.json` (counts only).
- Folder rule: prompts pin absolute paths inside Droid's folder. File reads never hit the permission handler, so a stray absolute-path Read can only be **detected** ([`test_privacy.py`](../sandbox_jobs/test_privacy.py)), not prevented.
- Daytona privacy mode also fails if planted markers appear anywhere in the job's backend folder except returned `insights.json`.

**Limits:** one-word titles can survive redaction; lengths/timestamps/model names are visible; `insights.json` quotes up to 3 example prompts (80 chars) by design; we have not further restricted sandbox outbound network ourselves.

## Audit, cleanup, replay

- [`GET /jobs/{id}/audit`](../backend/main.py) — sandbox id, create/delete times, every file in/out by name and size
- Delete in `finally`; ephemeral + auto-stop + time limit; labeled `app=promptprint` for [`cleanup_sandboxes.py`](../backend/cleanup_sandboxes.py)
- No keys in sandbox env (Droid's `FACTORY_API_KEY` stays on the host)
- `py start.py --replay` with Daytona key set replays saved Daytona runs including audit

## Measured sandbox timings (live smoke)

Upload sandbox work ~8 s of active steps (create, pip, upload, redact, run, download, delete) while the sandbox waits ~minutes for Droid on the backend. Repair sandbox life ~2 s of cleaning around a ~60 s Droid turn.

## Quick commands

```sh
py start.py                                   # Daytona mode if DAYTONA_API_KEY set
py backend/try_sandbox_flow.py                # stand-in flow in a sandbox
py backend/cleanup_sandboxes.py --dry-run
PRIVACY_MODE=daytona py -m pytest sandbox_jobs/test_privacy.py
```
