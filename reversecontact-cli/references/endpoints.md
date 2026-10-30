# Endpoint Reference

All requests go through the `rc` CLI to the Reverse Contact gateway API (`RC_BASE_URL`, default `https://api.reversecontact.com`).

Credit numbers come from shared V2 pricing. Enrich always returns the full profile; `--full-profile` is still accepted but ignored.

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
**Credits:** cached 1 if found / 0 on 404; live 2 (async).

| Flag | Body / behavior |
|------|-----------------|
| `--url` | `url` |
| `--id` | `id` (cached or live) |
| `--live` | live route; requires exactly one of `--url` or `--id` |
| `--webhook-url` | `webhookUrl` (requires `--live`) |
| `--wait` | poll GET `/v2/webhooks/:id` until terminal (requires `--live`; budget `--timeout`) |
| `--fields` | projection on entity |
| `--json` / `--api-key` / `--timeout` / `--quiet` | common |

Exactly one of `--url` or `--id` is required in both cached and live modes. Without `--wait`, live mode prints/returns the task (`webhookId`) and exits 0. Prefer default (no `--wait`) for agents; re-check with `rc task <id>`.

**Response (cached):** entity at `data.person` / `data.company`, or sometimes as `data` root. 404 → exit 0, free.

**Response (live ack):** `data.webhookId`, `data.status`, optional `data.pollUrl` (ack only; not on poll).

**Response (live + `--wait`):** CLI polls `GET /v2/webhooks/:webhookId` and, on `succeeded`, unwraps `data.result` into the same entity path as cached fetch (`data.person` / `data.company`), so `--fields` and human renderers apply. On `errored` → exit 1 (`task_errored`). Poll budget exceeded → exit 4 (`poll_timeout`).

---

## `rc enrich person`

**API:** `POST /v2/enrich/persons`  
**Credits:** 2 if found (full profile included), 0 on miss

| Flag | Body field | Notes |
|------|------------|-------|
| `--email` | `email` | Work email |
| `--first-name` | `firstName` | |
| `--last-name` | `lastName` | |
| `--company-domain` | `companyDomain` | e.g. `acme.com` |
| `--company-name` | `companyName` | |
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
rc fetch person --url https://linkedin.com/in/jane
rc enrich person --email unknown@example.com --fail-on-not-found
```

---

## `rc enrich company`

**API:** `POST /v2/enrich/companies`  
**Credits:** 2 if found (full profile included), 0 on miss

| Flag | Body field | Notes |
|------|------------|-------|
| `--domain` | `domain` | Required |
| `--fail-on-not-found` | (behavior) | Exit 1 on 404 |
| `--fields` | (projection) | |

Entity at `data.company`. For company LinkedIn URL/id use `rc fetch company` instead (1 credit).

```bash
rc enrich company --domain acme.com
rc enrich company --domain acme.com --fields companyName,industry,employeesCount,description
```

---

## `rc search people`

**API:** `POST /v2/search/people` (search) · `POST /v2/search/people/count` (free total)
**Credits:** 1 per profile returned; count is 0 (free)

The CLI first POSTs the filters to `/v2/search/people/count` (free, stable total for the same filters), then runs the search. Cursor pagination: pass the previous `metadata.nextCursor` back as `--cursor`; omit it for the first page. `--limit` maps to `perPage` (1-100, API default 25). There is no `page` parameter.

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
| `--connections-count` | `connectionsCount` | `min-max`, `min-` or `-max`; one bound is enough (`501-`) |
| `--country` | `location.country` | comma multi |
| `--country-code` | `location.countryCode` | comma multi |
| `--area` | `location.area` | comma multi; zone ids (`MR:EMEA`, `MC:DACH`, `IG:EU`…), added to the countries, listed by `rc search fields people` |
| `--region` | `location.region` | comma multi |
| `--city` | `location.city` | comma multi |
| `--max-data-age-date` | `maxDataAgeDate` | ISO date-time |
| `--cursor` | `cursor` | opaque; from previous `metadata.nextCursor`, omit for the first page |
| `--limit` | `perPage` | omit → API default 25, max 100 |
| `--fields` | (projection) | per result item |

### Validation

At least one filter (pagination alone is not a filter). Filters AND-combined; multi-value title OR-matched. Invalid range/int → exit 2 (`invalid_size`).

### Response

Full profiles (same shape as `rc fetch person`) at `data.data`; page metadata at `data.metadata` (`perPage`, `count`, `nextCursor`). The stable total matching the filters comes from the free count endpoint (`data.count`), not from the search response. Top-level `quotas.creditsConsumed` equals the number of profiles delivered (1 credit per profile).

```bash
rc search people --title CTO --company-name Acme
rc search people --country France,Germany --city Paris --title "Engineering Manager" --limit 50 --cursor "$NEXT_CURSOR"
rc search people --open-to-work --company-industry "Software Development" --fields firstName,lastName,currentCompanyName
```

**Schema guard:** `rc search fields people` lists server filters (0 credits). CLI covers all public search body fields except internal `issuer`.

---

## `rc search organizations`

**API:** `POST /v2/search/organizations` (search) · `POST /v2/search/organizations/count` (free total)
**Credits:** 1 per organization returned; count is 0 (free)

Successor of `rc search companies` (as `rc search people` succeeded the person search). The CLI first POSTs the filters to `/v2/search/organizations/count` (free, stable total for the same filters), then runs the search. Cursor pagination: pass the previous `metadata.nextCursor` back as `--cursor`; omit it for the first page. `--limit` maps to `perPage` (1-100, API default 25). There is no `page` parameter.

| Flag | Body field | Notes |
|------|------------|-------|
| `--name` | `companyName` | |
| `--domain` | `companyDomain` | e.g. `acme.com` |
| `--industry` | `industry` | comma multi, OR match |
| `--specialities` | `specialities` | comma multi, OR match |
| `--size` | `employeeCountRange` | `min-max`, `min-` or `-max` → `{min?,max?}` |
| `--founded-year` | `foundedYear` | `min-max`, `min-` or `-max` |
| `--followers-count` | `followersCount` | `min-max`, `min-` or `-max` |
| `--country` | `location.country` | comma multi |
| `--country-code` | `location.countryCode` | comma multi |
| `--area` | `location.area` | comma multi; headquarters zone ids, added to the countries, listed by `rc search fields organizations` |
| `--region` | `location.region` | comma multi |
| `--city` | `location.city` | comma multi |
| `--max-data-age-date` | `maxDataAgeDate` | ISO date-time; narrows only, never the only filter |
| `--cursor` | `cursor` | opaque; from previous `metadata.nextCursor`, omit for the first page |
| `--limit` | `perPage` | omit → API default 25, max 100 |
| `--fields` | (projection) | per result item, e.g. `companyName,headquarter.city` |

### Validation

At least one filter besides `--max-data-age-date` (pagination alone is not a filter), else exit 2 (`missing_input`). Filters AND-combined; multi-value industry and specialities OR-matched. Invalid or reversed range, invalid int → exit 2 (`invalid_size`).

### Response

Full organization profiles at `data.data`; page metadata at `data.metadata` (`perPage`, `count`, `nextCursor`, `null` on the last page). The stable total comes from the free count endpoint (`data.count`). Top-level `quotas.creditsConsumed` equals the number of organizations delivered (1 credit per organization).

Row fields: `id`, `companyName`, `linkedinId`, `publicId`, `linkedinUrl`, `employeesCount`, `followersCount`, `employeeCountRange` (`start`/`end`), `websiteUrl`, `tagline`, `description`, `industry`, `phone`, `specialities[]`, `headquarter` (`street1`, `street2`, `postalCode`, `city`, `geographicArea`, `country`), `logoUrl`, `foundedOn.year`, `locations[]`, `backgroundUrl`.

Human output: one line per organization with name, industry, employees, headquarters city and country, website.

```bash
rc search organizations --industry "Software Development" --size 51-200 --country-code FR
rc search organizations --specialities "SaaS,AI" --founded-year 2015- --limit 50 --cursor "$NEXT_CURSOR"
rc search organizations --domain stripe.com --fields companyName,employeesCount,headquarter.city
```

**Schema guard:** `rc search fields organizations` lists server filters (0 credits). CLI covers all public search body fields except internal `issuer`.

---

## `rc search companies` (deprecated)

**API:** `POST /v2/search/companies`  
**Credits:** 1 per page

Deprecated in favour of `rc search organizations`; the endpoint is still served. In human mode the CLI prints a one-line deprecation notice on stderr; `--json` output is untouched (no notice on stdout or stderr). New scripts should use `rc search organizations`.

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
| `--page` | `page` | default 1, max 50 |
| `--limit` | `perPage` | default 100, max 100 |

At least one filter required. Response envelope: results at `data.data`, page metadata at `data.metadata` (`currentPage`, `pageNumber`, `perPage`, `total`). Companies keep `page`/`perPage` pagination and cost 1 credit per call.

**Note on `employeeCountRange` in results:** response uses `start`/`end` (not `min`/`max`). For the 10,001+ bucket, `end` may be less than `start` (LinkedIn sentinel).

```bash
rc search companies --industry "Software Development" --size 51-200 --city Austin
rc search companies --name Stripe --country "United States"
```

---

## `rc search fields people` / `rc search fields organizations` / `rc search fields companies`

**API:** `GET /v2/search/people/available-fields` · `GET /v2/search/organizations/available-fields` · `GET /v2/search/companies/available-fields` (deprecated)
**Credits:** 0

No extra flags beyond common. Use to stay aligned with server schema. `companies` keeps returning the catalog of the deprecated `rc search companies`; use `organizations` for `rc search organizations`.

---

## `rc contact email`

**API:** `POST /v2/contact/email`  
**Credits:** only when an email is found (async), billed on the emails found, not on `--type`: 3 for a professional email, 5 for a personal one, so `--type any` costs 3, 5 or 8 (8 only when it finds both). `personal` and `any` debit 5 when the job starts; `any` is then adjusted to what was found.

| Flag | Body |
|------|------|
| `--url` | `url` |
| `--id` | `id` (person id, `prs_…`) |
| `--full-name` | `fullName` |
| `--first-name` | `firstName` |
| `--last-name` | `lastName` |
| `--company-domain` | `companyDomain` |
| `--type` | `emailType`: `professional` (default when omitted), `personal`, or `any` for either |
| `--webhook-url` | `webhookUrl` |

Need `--url`, or `--id`, or `--full-name`, or name parts. `--url` and `--id` together exit 2 (`conflicting_flags`). Pass `--type personal` only when the user wants a personal address; `any` stops at the first email found and can return both kinds. Returns a job acknowledgement with `webhookId`; the result is an array of `{ value, type }` (`type: "professional"` or `"personal"`), delivered through the configured webhook callback or `rc task <taskId>`.

---

## `rc contact phone`

**API:** `POST /v2/contact/phone`  
**Credits:** 20 only when a phone number is found (async). Returns HTTP 410 (`endpoint_unavailable`) when phone finding is not enabled for the workspace.

| Flag | Body |
|------|------|
| `--url` | `url` |
| `--id` | `id` (person id, `prs_…`) |
| `--full-name` | `fullName` |
| `--first-name` | `firstName` |
| `--last-name` | `lastName` |
| `--company-domain` | `companyDomain` |
| `--webhook-url` | `webhookUrl` |

Need `--url`, or `--id`, or `--full-name`, or name parts. `--url` and `--id` together exit 2 (`conflicting_flags`). Returns a job acknowledgement with `webhookId`; the result is an array of `{ value, type }` (e.g. `type: "mobile"`), delivered through the configured webhook callback or `rc task <taskId>`.

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

`data` is null; read `quotas.workspace` (`credits`, `dailyLimit`, `minuteRateLimit`, `hasUnlimitedCredits`). `quotas.key.creditLimits` holds the credit limits of the API key (`daily`, `monthly`, `total`, each `{ limit, used, left, nextReset }` or null when the key has no such limit).

```bash
rc usage --json | jq '.quotas.workspace.credits.left'
rc usage --json | jq '.quotas.workspace.minuteRateLimit.left'
rc usage --json | jq '.quotas.workspace.dailyLimit'
rc usage --json | jq '.quotas.key.creditLimits.daily.left'
```

`rc auth status --json` returns the same workspace quotas plus key `source` and masked key.
