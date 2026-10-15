---
name: reversecontact-cli
description: Enrich, fetch, search and find contact emails and phone numbers for people and companies via the Reverse Contact rc CLI. Use when the user wants to enrich from email or name, fetch by LinkedIn URL or id, search people or companies, check profile freshness for free, resolve contact emails or phone numbers, poll async live jobs, or inspect credits. Requires the rc binary and a Reverse Contact API key.
compatibility: Requires the rc binary (macOS/Linux `curl -fsSL https://app.reversecontact.com/cli/install.sh | sh` or `brew install visum-run/tap/rc`, Windows `powershell -c "irm https://app.reversecontact.com/cli/install.ps1 | iex"`) and network access
metadata:
  author: reversecontact
  version: "0.3.0"
---

<!-- Published as reversecontact-cli/ in visum-run/skills by the release workflow (source: apps/cli/skill/). Install: npx skills add visum-run/skills -->

## Setup

### Non-interactive authentication (preferred for agents)

```bash
# Option 1: environment variable (no disk write, ephemeral)
export RC_API_KEY=rc_your_key_here

# Option 2: store key in config file
rc auth login rc_your_key_here

# Option 3: pipe from secret manager
echo "$RC_API_KEY" | rc auth login --with-token
```

Verify before any operation:

```bash
rc auth status --json
# exit 3 = not authenticated or invalid key; exit 0 = ok
```

JSON contains `source` (flag | env | config), masked `key`, and `workspace` credit balance (plus `dailyLimit` and `minuteRateLimit` when present).

**Human-only flow:** `rc auth login --browser` starts the device flow (verification URL + user code, then poll). Use `--no-open` in headless environments to print the URL without launching a browser. Agents should prefer `RC_API_KEY` or `--with-token`. Failure modes (denied, expired, timeout) exit 3.

**Remove stored key:** `rc auth logout`

### Key resolution order

1. `--api-key <key>` flag
2. `RC_API_KEY` environment variable
3. Config file written by `rc auth login`

### Config file path

| Platform | Path |
|----------|------|
| Linux / macOS (XDG) | `$XDG_CONFIG_HOME/reversecontact/config.json` if set, else `~/.config/reversecontact/config.json` |
| Windows | `%APPDATA%\reversecontact\config.json` |

File mode `0600` on POSIX. Safe to inspect or delete for cleanup.

### Environment variables

| Variable | Effect |
|----------|--------|
| `RC_API_KEY` | API key (overrides config) |
| `RC_BASE_URL` | Gateway base URL (default `https://api.reversecontact.com`) |
| `RC_OUTPUT` | Force `json` or `human` without `--json` |
| `RC_DEBUG=1` | Print stack traces on errors |
| `RC_NO_UPDATE_CHECK=1` | Disable `rc version` update check |

---

## Output contract for agents

**JSON is automatic when stdout is not a TTY** (scripts and pipes). `--json` forces JSON always.

| Stream | Content |
|--------|---------|
| stdout | Full API response as pretty-printed JSON |
| stderr | Errors as `{"error":{"code":"...","message":"...","hint":"..."}}` |

**Not-found is not an error.** A 404 on enrich/fetch exits 0 with `success: false` (0 credits). Use `--fail-on-not-found` on enrich only to get exit 1.

**Limit fields** with `--fields` (comma-separated dotted paths).

### Exit codes

| Code | Meaning |
|------|---------|
| 0 | Success (including not-found when `--fail-on-not-found` is absent) |
| 1 | API error (4xx/5xx after client handling, including 402/403 insufficient credits, task failure) |
| 2 | Usage error (bad flags, missing required input, conflicting flags) |
| 3 | Authentication error only: missing key or HTTP 401 invalid key |
| 4 | Network error (request timeout, unreachable host, **or** live-job poll timeout) |

**Important:** HTTP 402/403 map to exit **1** (`insufficient_credits`), not exit 3. Exit 3 is reserved for missing/invalid API key (401).

### Error codes (`error.code`)

| Code | Exit | When |
|------|------|------|
| `missing_api_key` | 3 | No key from flag, env, or config |
| `invalid_api_key` | 3 | HTTP 401 |
| `insufficient_credits` | 1 | HTTP 402 or 403 |
| `rate_limited` | 1 | HTTP 429 after retries |
| `server_error` | 1 | HTTP 5xx |
| `api_error` | 1 | Other HTTP 4xx from API, or malformed live response |
| `task_errored` | 1 | Async job finished with status `errored` |
| `timeout` | 4 | Single request timed out |
| `network_error` | 4 | Cannot reach base URL |
| `poll_timeout` | 4 | `--wait` / `rc task --wait` budget exhausted before terminal status |
| `missing_identifying_field` | 2 | Enrich person / contact email or phone missing identifiers |
| `missing_domain` | 2 | Enrich company without `--domain` |
| `missing_input` | 2 | Required flag/positional missing, or search with no filter |
| `invalid_size` | 2 | Bad range or positive-int flag value |
| `invalid_value` | 2 | Unrecognized boolean flag token (e.g. `--premium=banane`) |
| `conflicting_flags` | 2 | Mutually exclusive flags (e.g. `--url` and `--id`, `--wait` without `--live`) |
| `empty_token` | 2 | `--with-token` got empty stdin |
| `no_tty` | 2 | Interactive login without a TTY |
| `cancelled` | 2 | Interactive login cancelled |
| `device_code_failed` | 1 | Browser login could not start |
| `login_denied` | 3 | User denied browser login |
| `login_expired` | 3 | Browser login code expired |
| `login_timeout` | 3 | Browser login poll timed out |
| `unexpected_error` | 1 | Non-CliError thrown |

### HTTP retries

On HTTP 429 the client retries up to **3 attempts** with exponential backoff (base **1s**: 1s, then 2s). Factor this into agent `--timeout` budgets for high-throughput loops.

---

## Choosing the right command (credit strategy)

| Situation | Prefer | Cost (success) |
|-----------|--------|----------------|
| You have a LinkedIn URL or internal id (`prs_…` / `com_…`) | `rc fetch` | **1** credit cached |
| You only have email / name / domain to resolve | `rc enrich` | **2** credits, full profile included |
| Want to know if a profile exists / how fresh it is before paying | `rc check` | **0** credits |
| Discover server-side search filters (avoid doc drift) | `rc search fields people\|companies` | **0** credits |
| Need a fresh scrape | `rc fetch … --live` | **2** credits live; async |

**Trial plans** cannot call live routes (`--live`). The API returns access denied; upgrade required. Cached fetch/enrich/search/check remain available subject to plan limits.

**Live jobs are async.** Target latency is under 15 seconds, but jobs can take tens of minutes. Default without `--wait` returns a `taskId` (`webhookId`) immediately: preferred for agents. Poll later with `rc task <taskId>`, or use `--wait` only with a conscious `--timeout` (seconds; default 30, which is often too short for slow live jobs).

---

## Commands and credit costs

| Command | Credits | Required input |
|---------|---------|----------------|
| `rc auth login [key]` | 0 | key (arg, stdin, browser) |
| `rc auth status` | 0 | none |
| `rc auth logout` | 0 | none |
| `rc check person\|company` | **0** | `--url` **or** `--id` |
| `rc fetch person\|company` | 1 cached / 0 on 404; 2 live | `--url` or `--id` (exactly one in cached or live mode) |
| `rc enrich person` | 2 found (full profile), 0 miss | at least one identifying field |
| `rc enrich company` | 2 found (full profile), 0 miss | `--domain` |
| `rc search people` | 1 per profile delivered (the count it runs first is free) | at least one filter |
| `rc search companies` | 1 per page | at least one filter |
| `rc search fields people\|companies` | **0** | none |
| `rc contact email` | 3 when an email is found (async job) | `--url`, or `--full-name`, or name parts (+ optional domain) |
| `rc contact phone` | 20 when a phone number is found (async job) | `--url`, or `--full-name`, or name parts (+ optional domain) |
| `rc task <taskId>` | 0 | task id from a live job |
| `rc usage` | 0 | none |
| `rc version` | 0 | none |

### Common flags (most commands; not `rc version`)

| Flag | Description |
|------|-------------|
| `--json` | Force JSON output |
| `--api-key <key>` | Override stored/env key |
| `--quiet` | Suppress spinners and credit hints |
| `--timeout <seconds>` | Request (and poll budget with `--wait`) timeout; default 30 |
| `--fields <paths>` | Comma-separated dotted projection |

### `rc auth login` flags

| Flag | Description |
|------|-------------|
| `--with-token` | Read key from stdin |
| `--browser` | Device-flow browser login |
| `--no-open` | With `--browser`: print URL only (headless/agent) |

### `rc check person|company`

| Flag | API field |
|------|-----------|
| `--url` | `url` |
| `--id` | `id` |

Exactly one of `--url` or `--id`. Free existence / last-update check.

### `rc fetch person|company`

| Flag | API field / behavior |
|------|----------------------|
| `--url` | `url` |
| `--id` | `id` (cached or live) |
| `--live` | POST `…/live` (async; accepts either `--url` or `--id`; Trial blocked) |
| `--webhook-url` | `webhookUrl` (requires `--live`) |
| `--wait` | Poll until terminal (requires `--live`; budget = `--timeout`) |

Exactly one of `--url` or `--id` in cached mode. Prefer this over enrich when you already have a URL or id (1 credit vs 2).

### `rc enrich person`

| Flag | API field |
|------|-----------|
| `--email` | `email` |
| `--first-name` | `firstName` |
| `--last-name` | `lastName` |
| `--company-domain` | `companyDomain` |
| `--company-name` | `companyName` |
| `--fail-on-not-found` | exit 1 on 404 |

Does **not** accept a LinkedIn URL. Use `rc fetch person --url` for URL lookup (1 credit).

### `rc enrich company`

| Flag | API field |
|------|-----------|
| `--domain` | `domain` (required) |
| `--fail-on-not-found` | exit 1 on 404 |

### `rc search people`

| Flag | API field |
|------|-----------|
| `--first-name` | `firstName` |
| `--last-name` | `lastName` |
| `--headline` | `headline` |
| `--title` | `currentPositionTitle` (comma = multi, OR) |
| `--exclude-title` | `excludeCurrentPositionTitle` (comma) |
| `--company-name` | `currentCompanyName` |
| `--company-id` | `currentCompanyId` |
| `--company-linkedin-id` | `currentCompanyLinkedinId` |
| `--company-public-id` | `currentCompanyPublicId` |
| `--company-industry` | `currentCompanyIndustry` (comma) |
| `--company-size` | `currentCompanyEmployeeCountRange` (`min-max`) |
| `--open-to-work` / `--no-open-to-work` | `isOpenToWork` |
| `--premium` / `--no-premium` | `isPremium` |
| `--verified` / `--no-verified` | `isVerified` |
| `--followers-count` | `followersCount` (`min-max`) |
| `--country` | `location.country` (comma multi) |
| `--country-code` | `location.countryCode` (comma) |
| `--region` | `location.region` (comma) |
| `--city` | `location.city` (comma) |
| `--area` | `location.area` (comma) |
| `--max-data-age-date` | `maxDataAgeDate` (ISO date-time) |
| `--cursor` | `cursor` (previous `metadata.nextCursor`; omit for the first page) |
| `--limit` | `perPage` (API max 100) |

At least one filter required (pagination alone exits 2). Prefer `rc search fields people` when unsure which filters the server accepts.

### `rc search companies`

| Flag | API field |
|------|-----------|
| `--name` | `companyName` |
| `--domain` | `companyDomain` |
| `--industry` | `industry` (comma multi) |
| `--size` | `employeeCountRange` (`min-max`) |
| `--country` | `location.country` (comma) |
| `--country-code` | `location.countryCode` (comma) |
| `--region` | `location.region` (comma) |
| `--city` | `location.city` (comma) |
| `--max-data-age-date` | `maxDataAgeDate` |
| `--page` | `page` (API max 50) |
| `--limit` | `perPage` |

### `rc search fields people|companies`

GET available-fields catalog. Free. Use before inventing filter names.

### `rc contact email`

The command returns an acknowledgement with `webhookId`; deliver the result through the configured webhook callback.

### `rc contact phone`

Same flags and validation as `rc contact email`, targeting `POST /v2/contact/phone`. Costs 20 credits only when a phone number is found. The API returns HTTP 410 (`endpoint_unavailable`) when phone finding is not enabled for the workspace; fall back to `rc contact email`. Returns an acknowledgement with `webhookId`; the result is a list of `{ value, type }` entries (e.g. `type: "mobile"`), delivered through the webhook callback or `rc task <taskId>`.

### `rc task <taskId>`

| Flag | Behavior |
|------|----------|
| `--wait` | Poll until terminal (budget = `--timeout`) |

Poll JSON paths: `data.status`, `data.result` (entity when succeeded), `data.errorCode`, `data.deliveryStatus`. Missing task id → exit 2 (`missing_input`). See [endpoints.md](references/endpoints.md#rc-task-taskid).

---

## CLI vs MCP

The API also exposes an **MCP server** at `https://api.reversecontact.com/mcp`, with tools to enrich and search people and companies, count people, check profiles, find emails and phone numbers, and poll tasks and usage. Guide: https://docs.reversecontact.com/guides/mcp

| Prefer | When |
|--------|------|
| **MCP** | Conversational agent already connected to the MCP endpoint |
| **CLI** | Scripting, batch, pipes, exit-code control, field projection |

CLI-only (or stronger) capabilities: `--fields` projection, `--fail-on-not-found`, free `rc check`, full search filter surface (e.g. `isOpenToWork`, company size/ids, multi-value location), `rc contact`, standalone `rc fetch` by id, headless auth helpers (`--no-open`, `--with-token`).

---

## Before high-volume work

```bash
rc usage --json | jq '.quotas.workspace.credits.left'
```

If `hasUnlimitedCredits` is true, skip the credit gate. Also respect `dailyLimit` and `minuteRateLimit` when present. For URL-known leads, prefer `rc check` (free) then `rc fetch` (1 credit) over blind enrich (2).

---

## References

- [references/endpoints.md](references/endpoints.md): full flag tables, validation, response shapes
- [references/workflows.md](references/workflows.md): shell recipes (bulk enrich, search→fetch, credit-aware batch, live wait)
