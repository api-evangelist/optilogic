---
name: optilogic-run-and-monitor-a-job
description: Queue a Python model run on the Optilogic platform and follow it to completion, including the ledger and metrics an agent needs to explain what happened.
api: Optilogic REST API
base: https://api.optilogic.app/v0
operations:
  - POST /refreshApiKey
  - GET /workspaces
  - POST /{workspace}/job
  - GET /{workspace}/job/{jobKey}
  - GET /{workspace}/job/{jobKey}/ledger
  - GET /{workspace}/job/{jobKey}/metrics
  - DELETE /{workspace}/job/{jobKey}
generated: '2026-08-26'
method: generated
source: openapi/optilogic-rest-api-openapi.json
---

# Run and monitor an Optilogic job

The Optilogic REST API has **no operationIds**, so every step below names the HTTP method and path
verbatim from the published Swagger 2.0 document at `https://api-docs.optilogic.app/swagger-ui/swagger.json`.

## Before you start

- Base URL is `https://api.optilogic.app/v0`. The version is the **first** path segment and is validated
  before authentication — an unrecognised first segment returns `403 {"error":"Invalid Api Version: ..."}`,
  not a 404.
- Authenticate with the `X-API-KEY` request header on every call.
- A **free** Optilogic account has test access only and **cannot solve models**. If a job never leaves the
  queue, check the account tier before debugging the model.

## Steps

1. **Get a key.** `POST /refreshApiKey` with `X-USER-ID` + `X-USER-PASSWORD`, or with `X-REFRESH-KEY`.
   The returned API key is valid for **one hour**. An App key created in the Optilogic app does not expire —
   prefer that for an unattended agent so you are not re-authenticating mid-run.
2. **Pick a workspace.** `GET /workspaces` lists the workspaces on the account. Use the workspace **name**
   as the `{workspace}` path parameter.
3. **Confirm the module is there.** `GET /{workspace}/files` lists user files. All `directoryPath` values are
   relative to the workspace `/projects` directory — **do not** include `/projects` in the path string.
   `directoryPath: "pyomo-examples/models/diet-problem"`, `filePath: "diet-problem.py"`.
4. **Queue the job.** `POST /{workspace}/job` returns **202** with a `jobKey`. If a `requirements.txt` exists
   in the same `directoryPath`, the platform runs `pip install -r requirements.txt` before your module.
5. **Poll.** `GET /{workspace}/job/{jobKey}`. There are **no webhooks and no callbacks** in this contract, so
   completion is poll-only. There is also no documented rate limit and no `Retry-After` header — pace
   yourself; a few seconds between polls is reasonable for a solve.
6. **Explain the run.** `GET /{workspace}/job/{jobKey}/ledger` for the ordered log records and
   `GET /{workspace}/job/{jobKey}/metrics` (or `/metrics/stats` for the roll-up) for resource usage.
7. **Stop it if you must.** `DELETE /{workspace}/job/{jobKey}` stops a queued or running job.

## Rules an agent must follow

- **Queueing a job is not idempotent.** There is no `Idempotency-Key` on `POST /{workspace}/job`. A retry
  after a timeout queues a **second** job. Poll `GET /{workspace}/jobs` before retrying.
- **Stopping is not undoing.** `DELETE /{workspace}/job/{jobKey}` halts execution; files written and
  database rows changed by the partial run stay changed. No window or retention is published.
- **Errors are a vendor envelope, not RFC 9457.** Every failure returns
  `{"result":"error","error":"<reason>","correlationId":"<uuid>"}`. Log the `correlationId` — it is the only
  tracing identifier this API emits, and it appears on error bodies only, never as a response header.
- `401` means the key is missing or the hour expired. `403` means the account tier does not entitle the call.

## Batch variants

- `POST /{workspace}/jobBatch/jobify` queues a list of jobs.
- `POST /{workspace}/jobBatch/backToBack` runs a list of Python modules in sequence.
- Both have `searchNRun` siblings that take search terms instead of an explicit list.
- Reversal is per job: `DELETE /{workspace}/job/{jobKey}` for each key returned.
