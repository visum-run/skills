# Endpoint Reference

All requests go through the `rc` CLI to the ReverseContact gateway API (`RC_BASE_URL`, default `https://api.reversecontact.com`).

Credit numbers come from shared V2 pricing. `--full-profile` on enrich adds `FULL_PROFILE_EXTRA_CREDITS` (+1) when a match is found.

---

## `rc check person` / `rc check company`

**API:** `POST /v2/fetch/persons/check` · `POST /v2/fetch/companies/check`  
**Credits:** 0

| Flag | Body | Notes |
|------|------|-------|
| `--url` | `url` | LinkedIn URL |
| `--id` | `id` | `prs_…` or `com_…` |

Exactly one of `--url` or `--id`. Response `data` describes existence and last update (see human renderer fields: found, ids, update dates).

---

## `rc fetch person` / `rc fetch company`

**API (cached):** `POST /v2/fetch/persons` · `POST /v2/fetch/companies`  
**API (live):** `POST /v2/fetch/persons/live` · `POST /v2/fetch/companies/live`  
**Credits:** cached 1 if found / 0 on 404; live 2 (async). Trial plans cannot use `--live`.

| Flag | Body / behavior |
|------|-----------------|
| `--url` | `url` |
| `--id` | `id` (cached only; conflicts with `--live`) |
| `--live` | live route; requires `--url` |
| `--webhook-url` | `webhookUrl` (requires `--live`) |
| `--wait` | poll GET `/v2/webhooks/:id` until terminal (requires `--live`; budget `--timeout`) |
| `--fields` | projection on entity |
| `--json` / `--api-key` / `--timeout` / `--quiet` | common |

Exactly one of `--url` or `--id` in cached mode. Without `--wait`, live mode prints/returns the task (`webhookId`) and exits 0. Prefer default (no `--wait`) for agents; re-check with `rc task <id>`.

**Response (cached):** entity at `data.person` / `data.company`, or sometimes as `data` root. 404 → exit 0, free.

**Response (live ack):** `data.webhookId`, `data.status`, optional `data.pollUrl` (ack only; not on poll).

**Response (live + `--wait`):** CLI polls `GET /v2/webhooks/:webhookId` and, on `succeeded`, unwraps `data.result` into the same entity path as cached fetch (`data.person` / `data.company`), so `--fields` and human renderers apply. On `errored` → exit 1 (`task_errored`). Poll budget exceeded → exit 4 (`poll_timeout`).

---

## `rc enrich person`

**API:** `POST /v2/enrich/persons`  
**Credits:** 2 if found, 3 with `--full-profile` if found, 0 on miss

| Flag | Body field | Notes |
|------|------------|-------|
| `--email` | `email` | Work email |
| `--first-name` | `firstName` | |
| `--last-name` | `lastName` | |
| `--company-domain` | `companyDomain` | e.g. `acme.com` |
| `--company-name` | `companyName` | |
| `--full-profile` | `fullProfile` | +1 credit when found |
| `--fail-on-not-found` | (behavior) | Exit 1 on 404 |
| `--fields` | (projection) | |
| `--json` / `--api-key` / `--timeout` / `--quiet` | common | |

### Validation

At least one of `--email`, `--first-name`, `--last-name`, `--company-domain`, `--company-name`. Empty call exits 2 (`missing_identifying_field`).

**No LinkedIn URL on this route.** The enrich person DTO does not accept a profile URL. For URL (or id) lookup use `rc fetch person --url` / `--id` (1 credit cached).

### Response shape

Entity at `data.person`. Not-found: `success: false`, exit 0 (free). Credits only on match.

`data.alternativePersons` holds lower-scoring candidates when the match is ambiguous (common with email or name+company resolution). Often empty for high-confidence single matches.

```json
{
  "success": true,
  "data": {
    "person": {
      "id": "prs_9f8e7d6c5b4a",
      "publicId": "jane-doe",
      "linkedinUrl": "https://www.social.com/in/jane-doe",
      "firstName": "Jane",
      "lastName": "Doe",
      "headline": "VP of Marketing @ Acme Corp",
      "currentPositionTitle": "VP of Marketing",
      "currentCompanyId": "cmp_12345678",
      "currentCompanyName": "Acme Corp",
      "updateDate": "2026-04-07T14:22:17.271Z",
      "location": {
        "city": "San Francisco",
        "country": "United States",
        "countryCode": "US"
      }
    },
    "alternativePersons": []
  },
  "error": null,
  "metadata": { "requestId": "req_xxx", "executionTimeMs": 1240 },
  "quotas": {
    "creditsConsumed": 2,
    "workspace": {
      "id": "ws_abc123",
      "credits": { "total": 10000, "used": 4535, "left": 5465 },
      "dailyLimit": null,
      "minuteRateLimit": { "limit": 60, "used": 8, "left": 52, "nextReset": "..." },
      "hasUnlimitedCredits": false
    }
  }
}
```

### Examples

```bash
rc enrich person --email jane@acme.com
rc enrich person --first-name Jane --last-name Doe --company-domain acme.com \
  --fields linkedinUrl,currentPositionTitle
rc enrich person --email jane@acme.com --full-profile
rc fetch person --url https://linkedin.com/in/jane
rc enrich person --email unknown@example.com --fail-on-not-found
```

---

## `rc enrich company`

**API:** `POST /v2/enrich/companies`  
**Credits:** 2 if found, 3 with `--full-profile` if found, 0 on miss

| Flag | Body field | Notes |
|------|------------|-------|
| `--domain` | `domain` | Required |
| `--full-profile` | `fullProfile` | +1 credit when found |
| `--fail-on-not-found` | (behavior) | Exit 1 on 404 |
| `--fields` | (projection) | |

Entity at `data.company`. For company LinkedIn URL/id use `rc fetch company` instead (1 credit).

```bash
rc enrich company --domain acme.com
rc enrich company --domain acme.com --full-profile --fields name,industry,employeesCount
```

---

## `rc search people`

**API:** `POST /v2/search/persons`  
**Credits:** 1

| Flag | Body field | Notes |
|------|------------|-------|
| `--first-name` | `firstName` | |
| `--last-name` | `lastName` | |
| `--headline` | `headline` | |
| `--title` | `currentPositionTitle` | comma multi, OR match |
| `--exclude-title` | `excludeCurrentPositionTitle` | comma multi |
| `--company-name` | `currentCompanyName` | |
| `--company-id` | `currentCompanyId` | |
| `--company-linkedin-id` | `currentCompanyLinkedinId` | |
| `--company-public-id` | `currentCompanyPublicId` | |
| `--company-industry` | `currentCompanyIndustry` | comma multi |
| `--company-size` | `currentCompanyEmployeeCountRange` | `min-max` → `{min,max}` |
| `--open-to-work` / `--no-open-to-work` | `isOpenToWork` | only if flag present on argv |
| `--premium` / `--no-premium` | `isPremium` | |
| `--verified` / `--no-verified` | `isVerified` | |
| `--followers-count` | `followersCount` | `min-max` |
| `--country` | `location.country` | comma multi |
| `--country-code` | `location.countryCode` | comma multi |
| `--region` | `location.region` | comma multi |
| `--city` | `location.city` | comma multi |
| `--area` | `location.area` | comma multi |
| `--max-data-age-date` | `maxDataAgeDate` | ISO date-time |
| `--page` | `page` | default 1 |
| `--limit` | `perPage` | omit → API default 100, max 100 |
| `--fields` | (projection) | per result item |

### Validation

At least one filter (not pagination alone). Filters AND-combined; multi-value title OR-matched. Invalid range/int → exit 2 (`invalid_size`).

### Response

Nested envelope: `data.data` (array), `data.metadata` (pagination). Top-level `quotas` for credits.

```bash
rc search people --title CTO --company-name Acme
rc search people --country France,Germany --city Paris --title "Engineering Manager" --page 2 --limit 50
rc search people --open-to-work --company-industry Software --fields firstName,lastName,linkedinUrl
```

**Schema guard:** `rc search fields people` lists server filters (0 credits). CLI covers all public search body fields except internal `issuer`.

---

## `rc search companies`

**API:** `POST /v2/search/companies`  
**Credits:** 1

| Flag | Body field | Notes |
|------|------------|-------|
| `--name` | `companyName` | |
| `--domain` | `companyDomain` | |
| `--industry` | `industry` | comma multi |
| `--size` | `employeeCountRange` | `min-max` |
| `--country` | `location.country` | comma multi |
| `--country-code` | `location.countryCode` | |
| `--region` | `location.region` | |
| `--city` | `location.city` | |
| `--max-data-age-date` | `maxDataAgeDate` | |
| `--page` / `--limit` | `page` / `perPage` | same as people |

At least one filter required. Response envelope same as people search.

**Note on `employeeCountRange` in results:** response uses `start`/`end` (not `min`/`max`). For the 10,001+ bucket, `end` may be less than `start` (LinkedIn sentinel).

```bash
rc search companies --industry "Software Development" --size 51-200 --city Austin
rc search companies --name Stripe --country "United States"
```

---

## `rc search fields people` / `rc search fields companies`

**API:** `GET /v2/search/persons/available-fields` · `GET /v2/search/companies/available-fields`  
**Credits:** 0

No extra flags beyond common. Use to stay aligned with server schema.

---

## `rc contact email`

**API:** `POST /v2/contact/email`  
**Credits:** 4 (async). Trial restricted.

| Flag | Body |
|------|------|
| `--url` | `url` |
| `--full-name` | `fullName` |
| `--first-name` | `firstName` |
| `--last-name` | `lastName` |
| `--company-domain` | `companyDomain` |
| `--webhook-url` | `webhookUrl` |

Need `--url`, or `--full-name`, or name parts. Returns job ack with `webhookId`; poll via `rc task`.

---

## `rc contact verify`

**API:** `POST /v2/contact/email/verify`  
**Credits:** 2 (async). Trial restricted.

| Flag | Body |
|------|------|
| `--email` | `email` (required) |
| `--webhook-url` | `webhookUrl` |

---

## `rc task <taskId>`

**API:** `GET /v2/webhooks/:taskId`  
**Credits:** 0

| Flag | Behavior |
|------|----------|
| `--wait` | poll until `succeeded` or `errored` (budget `--timeout`) |

Missing `taskId` → exit 2 (`missing_input`). Terminal `errored` → exit 1 (`task_errored`). Poll budget exceeded → exit 4 (`poll_timeout`).

### Poll response shape (`data` = PollWebhookOutput)

The gateway never puts the enrichment payload under `data.data`. Read these paths:

| Path | Meaning |
|------|---------|
| `data.webhookId` | Task id |
| `data.status` | `created` / `in_progress` / `succeeded` / `errored` |
| `data.eventType` | Live event type (e.g. person live) |
| `data.errorCode` | Failure code when `status` is `errored`; otherwise null |
| `data.deliveryStatus` | Webhook delivery failure reason when enrichment succeeded but push failed; otherwise null |
| `data.createdAt` | ISO timestamp |
| `data.result` | Enrichment payload when succeeded; null otherwise |

```bash
# Status only
rc task "$task_id" --json | jq -r '.data.status'

# Entity after success (shape depends on the job: person, company, contact, …)
rc task "$task_id" --json | jq '.data.result'

# Error detail
rc task "$task_id" --json | jq -r '.data.errorCode'
```

With `--fields`, projection is applied to the poll envelope (`data`), e.g. `--fields status,result,errorCode`.

Note: `rc fetch … --live --wait` is different: it unwraps `result` and prints the entity as if it were a cached fetch. `rc task` always returns the poll envelope.

---

## `rc usage`

**API:** `GET /v2/usage`  
**Credits:** 0

`data` is null; read `quotas.workspace` (`credits`, `dailyLimit`, `minuteRateLimit`, `hasUnlimitedCredits`).

```bash
rc usage --json | jq '.quotas.workspace.credits.left'
rc usage --json | jq '.quotas.workspace.minuteRateLimit.left'
rc usage --json | jq '.quotas.workspace.dailyLimit'
```

`rc auth status --json` returns the same workspace quotas plus key `source` and masked key.
