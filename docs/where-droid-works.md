# Where Droid works

Factory Droid runs **at runtime inside the product**, not as a developer tool. Uploads and capture failures trigger jobs. Output reaches users only after our own checks pass.

Overview: [root README](../README.md) and [Backend architecture](backend-architecture.md). How the extension reports a broken page and hot-loads the fix: [Extension architecture](extension-architecture.md#self-healing-selectors-the-mechanic-loop). Sandbox boundary: [Where Daytona works](where-daytona-works.md). Eval contract: [Try it with Droid](try-it-with-droid.md).

## How jobs start

One backend ([`backend/main.py`](../backend/main.py)) turns product events into Droid jobs via [`run_droid_job`](../sandbox_jobs/droid_runner.py):

- Opens a Droid SDK `Session` in the job's own folder
- Autonomy `OFF` so every write/command hits our permission handler
- Writes allowed only under `work/droid` and `work/output`
- Python commands only on files under `work/`
- Logs every message and tool call; never raises on a Droid error
- Up to 2 follow-up turns if a refused action ended the turn

Success is decided by **our checks on the files Droid wrote**, not by how Droid's turn ended.

| Job | Trigger | What Droid does | Output | How we verify |
| --- | --- | --- | --- | --- |
| **Ingestor** | `POST /upload` (new format; cache hit = 0 credits) | Inspects **redacted** export, writes `parser.py` + tests | `parser.py` → `prompts.jsonl` | [`validate_prompts.py`](../sandbox_jobs/lib/validate_prompts.py); cache after redacted + real runs pass |
| **Analyst** | Every upload after Ingestor | Writes `analyze.py` from fake prompts + aggregates; numbers from [`footprint.py`](../backend/footprint.py) | `insights.json` (totals + 3 tips) | Four checks: footprint SHA, schema, re-run identical, no hard-coded footprint literals |
| **Mechanic** | `POST /adapter-failure` | Searches **cleaned** page, writes fixed adapter, bumps version | Adapter JSON | [`check_adapter.py`](../sandbox_jobs/lib/check_adapter.py); keys unchanged; version exactly +1 |

Code entry points: [`ingest.py`](../sandbox_jobs/ingest.py), [`analyze.py`](../sandbox_jobs/analyze.py), [`repair_adapter.py`](../sandbox_jobs/repair_adapter.py). Runner smoke: [`try_runner.py`](../sandbox_jobs/try_runner.py).

## Measured cost (Daytona mode)

- Ingestor: 131–169 s / 31k–39k factory_credits per new format (0 on cache hit)
- Analyst: 176–270 s / 28k–60k credits
- Full upload: ~60k–100k credits
- Repair: 62–65 s / ~5k credits
- Latest live both-flows (`py smoke_test.py --live`): **65,302** credits total

## Guardrails (Droid side)

| Guardrail | What it does |
| --- | --- |
| Path-restricted permissions | Approve writes only in `work/droid` and `work/output`; refuse pipes, redirects, `python -c`, other programs |
| Fingerprinted `lib/` and `input/` | Job fails if either folder changes during the run |
| `footprint.py` sole number source | SHA-256 + literal scan + doubling sensitivity test |
| Never sees real chat text | Redacted export / fake prompts + `aggregates.json` only |
| No chat text in logs | Paths, tool names, capped assistant text; tool results by name only |
| Unattended | Declines questions; follow-ups after refusals |

## Why Droid is not inside Daytona

Our Daytona tier blocks Factory domains from the sandbox. Hybrid: Droid on the backend with redacted/fake data; scripts it wrote run on real data in the sandbox. Details: [Where Daytona works](where-daytona-works.md).

## Demo fallbacks

- Saved runs: [`backend/demo_runs/`](../backend/demo_runs/)
- `py start.py --replay` / `DEMO_REPLAY=1`
- `py smoke_test.py` without `--live` uses `DROID_SAVED_DIR` stand-ins; sandboxes and validators still run for real

## Known issues

- droid-sdk 0.5.0 cannot map Droid's `Script` tool → turn ends. Prompts tell Droid not to use it.
- SDK ↔ Factory connection errors have ended turns; checks still accept finished work.
- Analyst cost varies a lot (one run ~158k credits). Check remaining Factory credits before live rehearsals.
- `footprint.py` still has placeholder ranges.

## Quick commands

```sh
py sandbox_jobs/smoke_droid.py      # Droid answers OK
py sandbox_jobs/try_runner.py       # hello job + blocked write into lib/
py smoke_test.py                    # both flows, stand-ins, 0 credits
py smoke_test.py --live             # real Droid (~100k credits)
```
