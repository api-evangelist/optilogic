---
name: optilogic-authenticate-and-handle-errors
description: Obtain and refresh Optilogic credentials, choose between a one-hour API key and a non-expiring App key, and interpret every documented failure the API returns.
api: Optilogic REST API
base: https://api.optilogic.app/v0
operations:
  - POST /refreshApiKey
  - GET /secrets
  - GET /secret/{secretName}
  - POST /secret/{secretName}
  - PUT /secret/{secretName}
  - DELETE /secret/{secretName}
generated: '2026-08-26'
method: generated
source: openapi/optilogic-rest-api-openapi.json
---

# Authenticate to Optilogic and read its errors

## The credential

Every call carries `X-API-KEY`. There are two kinds of value that fit that header and they behave differently:

| Credential | How you get it | Lifetime |
|---|---|---|
| **API key** | `POST /refreshApiKey` with `X-USER-ID` + `X-USER-PASSWORD`, or with `X-REFRESH-KEY` | **one hour** |
| **App key** | Created in the Optilogic application UI | does not expire |

For an unattended agent, prefer the **App key**. An API key minted through `/refreshApiKey` will expire
mid-session and every subsequent call returns `401 {"error":"Header 'x-api-key' missing"}`-class failures.

There is no OAuth on this REST API. OAuth exists only on the separate MCP server at
`https://mcp.optilogic.app/mcp` (Keycloak realm `che`, dynamic client registration, S256 PKCE) — a different
product surface, not an alternative credential for these endpoints.

## Storing other credentials

The API has an account-scoped secret store, which is where a model that needs third-party credentials should
read them from rather than embedding them in a module:

- `GET /secrets` — list. `GET /secret/{secretName}` — read one.
- `POST /secret/{secretName}` — add. `PUT /secret/{secretName}` — change one or more fields.
- `DELETE /secret/{secretName}` — delete. This **is** the reversal, but there is no restore: a deleted secret
  is gone and no retention window is published.

## The error envelope

Not RFC 9457. Every failure is the same vendor shape:

```json
{ "result": "error", "error": "<human readable reason>", "correlationId": "<uuid>" }
```

`correlationId` is the only tracing identifier this API produces, and it appears **in the body of errors
only** — there is no request-id response header. Capture it on every non-2xx and quote it to
support@optilogic.com.

## Status codes, in the order you will actually meet them

- **403 `Invalid Api Version: '<segment>'`** — you omitted `/v0`. The gateway parses the first path segment as
  the version and rejects it *before* authentication, so this looks like an auth problem and is not one.
- **401 `Header 'x-api-key' missing`** — no key, or the one-hour API key expired. Re-mint or switch to an App key.
- **403** (62 operations) — authenticated but not entitled. A free account has test access only and cannot
  upload, download or solve. Upgrading is a billing action, not a retry.
- **400** (62 operations) — malformed or missing parameter.
- **404** (59 operations) — the named workspace, jobKey, storageName, file, column or secret does not exist.
  Enumerate with the matching list operation before assuming a bug.
- **412** (2 operations) — custom-column precondition failed. Validate, repair, retry.

## What is not there

There is **no 429 anywhere in this contract**, and no `X-RateLimit-*`, `RateLimit-*` or `Retry-After` header is
declared on any response. You get no backpressure signal at all, so implement your own pacing and treat an
unexplained failure burst as possible throttling rather than expecting the API to tell you.
