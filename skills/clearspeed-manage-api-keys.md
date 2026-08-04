---
name: Create and rotate Clearspeed API keys
description: Mint questionnaire-scoped Clearspeed API keys with least-privilege scopes and rotate them with zero downtime using the createApiKey and deleteApiKey operations.
api: openapi/clearspeed-integration-api-openapi.yml
generated: '2026-08-04'
method: generated
source: openapi/clearspeed-integration-api-openapi.yml + https://developer.clearspeed.com/api-keys
operations:
  - createApiKey
  - deleteApiKey
operationIds:
  - createApiKey
  - deleteApiKey
---

# Create and rotate Clearspeed API keys

Clearspeed API keys are **scoped to exactly one questionnaire** and carry an explicit
scope list. There is no list operation, so you must record every key you create.

## Bootstrapping

The API cannot mint your first key. An Admin creates it in the Clearspeed web app
(questionnaire → Integration page → API Keys → **+ Generate New**), selects scopes, and
copies the value — it is fully visible only at creation. Give that first key
`apikey:write` if you intend to manage keys programmatically afterwards.

Authenticate with the raw value, no `Bearer` prefix:
`Authorization: <your-api-key>`

Region hosts: `https://api.us.clearspeed.com/questionnaire` (US) or
`https://api.uk.clearspeed.com/questionnaire` (UK).

## Scopes

| Scope | Grants |
|---|---|
| `participant:write` | Create participants; update participant outcome |
| `participant:read` | Read participant data |
| `participant:delete` | Delete participants |
| `apikey:write` | Create API keys for the same questionnaire |
| `apikey:delete` | Delete API keys for the same questionnaire |

The `ApiKeyCreateRequest` schema in the OpenAPI enumerates only `participant:write`,
`apikey:write` and `apikey:delete`; the portal documents `participant:read` and
`participant:delete` as well. Grant the narrowest set that works — an integration that
only submits participants needs `participant:write` and nothing more.

## Step 1 — Create a key

`createApiKey` — `POST /tenants/{tenant_id}/questionnaires/{questionnaire_id}/apikeys`

Path: `tenant_id` and `questionnaire_id`, both UUIDs. Body: `key_name` (required,
human-readable) and `scopes` (required array).

A `201` returns `id`, `api_key`, `key_name`, `scopes`, `questionnaire_id`, `create_ts`
and `update_ts`. **The `api_key` value is returned in full only here.** Write it straight
into your secret store — there is no list or read operation to recover it, and every
later view is masked.

Name keys after their consumer (`ci-participant-writer`, not `admin-key`) so you can
retire one without auditing the others.

## Step 2 — Rotate with zero downtime

1. `createApiKey` a replacement with **identical scopes** and a new name.
2. Deploy the new value to the integration.
3. Verify traffic succeeds on the new key.
4. `deleteApiKey` the old one.

Never delete first. Deletion is immediate and irreversible — anything still holding the
old key starts getting `401 Invalid API key` on the next request.

## Step 3 — Delete a key

`deleteApiKey` — `DELETE /tenants/{tenant_id}/questionnaires/{questionnaire_id}/apikeys/{apikey}`

The `{apikey}` path parameter is the **raw `api_key` value**, not the `id` UUID returned
on creation. That means you must have retained the secret in order to revoke it — another
reason to store it at creation time. A `204` means it is gone.

## Errors

Key endpoints use a different, thinner envelope than the participant endpoints: a single
`error` string, with no code, status field or timestamp. Branch on HTTP status.

- **400** `key_name is required and cannot be null or empty` / `scope is required and
  cannot be null or empty` (the request field is `scopes`, plural — the message says
  `scope`) / `API key value is required` on delete.
- **401** `Authorization header is required` / `Invalid API key`.
- **403** `You are not authorized to perform this operation` — the calling key lacks
  `apikey:write`. On delete the message is `Invalid API key scope` — it lacks
  `apikey:delete`.
- **404** `API key not found or Invalid tenant for this questionnaire` — the key does not
  exist, or the questionnaire does not belong to that tenant.

## Operating notes

There is no list operation, no expiry, no last-used timestamp and no rotation reminder.
Maintain your own inventory of `id`, `key_name`, `scopes`, `questionnaire_id` and
`create_ts` at creation time, and schedule rotation yourself.

See `authentication/clearspeed-authentication.yml` and
`errors/clearspeed-problem-types.yml`.
