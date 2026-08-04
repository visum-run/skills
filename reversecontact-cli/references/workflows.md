# Workflow Recipes

Short shell snippets ready to adapt. Prefer JSON mode in pipes (automatic when stdout is not a TTY).

---

## 1. Enrich a list of emails

Reads emails from a file, enriches each, writes JSONL. Separates not-found (exit 0, free) from hard errors.

```bash
#!/usr/bin/env bash
set -euo pipefail

INPUT="emails.txt"
OUTPUT="enriched.jsonl"
ERRORS="errors.jsonl"
: > "$OUTPUT"
: > "$ERRORS"

while IFS= read -r email; do
  [[ -z "$email" ]] && continue

  set +e
  result=$(rc enrich person --email "$email" \
    --fields linkedinUrl,firstName,lastName,currentPositionTitle,currentCompanyName 2>/tmp/rc_err)
  exit_code=$?
  set -e

  if [[ $exit_code -eq 0 ]]; then
    found=$(echo "$result" | jq -r '.success')
    if [[ "$found" == "true" ]]; then
      echo "$result" | jq --arg email "$email" '.data.person + {_email: $email}' >> "$OUTPUT"
    else
      echo "{\"email\": \"$email\", \"found\": false}" >> "$ERRORS"
    fi
  else
    # exit 1 = API/credits; exit 3 = auth; exit 4 = network
    cat /tmp/rc_err >> "$ERRORS"
    echo "Error enriching $email (exit $exit_code), continuing..." >&2
  fi
done < "$INPUT"

echo "Done. Results: $OUTPUT  Errors/not-found: $ERRORS"
```

Key points:
- `--fields` keeps rows small.
- Not-found is exit 0 with `success: false` (0 credits). Do not set `--fail-on-not-found` in a bulk loop.
- Exit 1 includes insufficient credits (402/403), not only server faults.

---

## 2. Search then fetch (cheapest chain)

Search returns LinkedIn URLs and ids. Fetch by URL costs 1 credit vs enrich at 2. Optional free `rc check` gates freshness.

```bash
#!/usr/bin/env bash
set -euo pipefail

COMPANY="Stripe"
TITLE="VP of Engineering"
MAX=5

search_result=$(rc search people \
  --company-name "$COMPANY" \
  --title "$TITLE" \
  --limit 25 \
  --fields firstName,lastName,linkedinUrl,id)

total=$(echo "$search_result" | jq '.data.metadata.total')
echo "Found $total people matching '$TITLE' at $COMPANY"

echo "$search_result" \
  | jq -r --argjson max "$MAX" \
    '.data.data[:$max] | .[] | [.id // "", .linkedinUrl // "", .firstName, .lastName] | @tsv' \
  | while IFS=$'\t' read -r id url first last; do
    echo "Processing $first $last..."

    # Free existence / freshness gate when URL is known
    if [[ -n "$url" ]]; then
      rc check person --url "$url" --quiet >/dev/null || true
      rc fetch person --url "$url" \
        --fields linkedinUrl,currentPositionTitle,location.city,currentCompanyName \
        >> profiles.jsonl 2>/dev/null || true
    elif [[ -n "$id" ]]; then
      rc fetch person --id "$id" \
        --fields linkedinUrl,currentPositionTitle,location.city,currentCompanyName \
        >> profiles.jsonl 2>/dev/null || true
    fi
  done

echo "Profiles saved to profiles.jsonl"
```

Key points:
- Search: 1 credit for the page. Fetch: 1 credit per hit. Enrich would be 2 per hit.
- Prefer `--url` or `--id` from search results; only fall back to `rc enrich person` when neither is available.
- Discover filters with `rc search fields people` before hard-coding new criteria.

---

## 3. Credit-aware batch

Check balance first, stop when the reserve is hit. Uses live `quotas` on each response.

```bash
#!/usr/bin/env bash
set -euo pipefail

DOMAINS_FILE="domains.txt"
RESERVE=50
COST_PER_HIT=2   # enrich company = 2 when found

usage=$(rc usage --json)
credits_left=$(echo "$usage" | jq '.quotas.workspace.credits.left')
unlimited=$(echo "$usage" | jq '.quotas.workspace.hasUnlimitedCredits')

if [[ "$unlimited" == "true" ]]; then
  echo "Unlimited credits; no budget cap."
  max_enrichments=999999
else
  echo "Credits left: $credits_left"
  budget=$((credits_left - RESERVE))
  if [[ $budget -le 0 ]]; then
    echo "Insufficient credits (need more than $RESERVE in reserve). Aborting." >&2
    exit 1
  fi
  max_enrichments=$((budget / COST_PER_HIT))
  echo "Can afford up to $max_enrichments company enrichments."
fi

count=0
while IFS= read -r domain; do
  [[ -z "$domain" ]] && continue
  [[ $count -ge $max_enrichments ]] && { echo "Credit budget reached, stopping."; break; }

  set +e
  result=$(rc enrich company --domain "$domain" 2>/dev/null)
  exit_code=$?
  set -e

  if [[ $exit_code -eq 0 ]]; then
    found=$(echo "$result" | jq -r '.success')
    if [[ "$found" == "true" ]]; then
      echo "$result" | jq '.data.company' >> companies.jsonl
      count=$((count + 1))
      credits_after=$(echo "$result" | jq '.quotas.workspace.credits.left')
      echo "[$count/$max_enrichments] $domain enriched. Credits left: $credits_after"
      if [[ $credits_after -lt $RESERVE ]]; then
        echo "Credits below reserve ($RESERVE). Stopping." >&2
        break
      fi
    else
      echo "$domain: not found (0 credits charged)"
    fi
  else
    echo "Error on $domain (exit $exit_code), skipping." >&2
  fi
done < "$DOMAINS_FILE"

echo "Finished. $count companies enriched. Results: companies.jsonl"
```

Key points:
- `rc usage` costs 0 credits.
- Factor `dailyLimit` and minute rate limits from the same payload when present.
- 429s are retried up to 3 times with exponential backoff (base 1s) inside the client.

---

## 4. Live fetch without blocking (agent-friendly)

```bash
# Start live job (returns task id; do not use --wait in tight agent loops)
ack=$(rc fetch person --url https://linkedin.com/in/jane --live --json)
task_id=$(echo "$ack" | jq -r '.data.webhookId')

# Later (or in another process)
poll=$(rc task "$task_id" --json)
status=$(echo "$poll" | jq -r '.data.status')
# succeeded → entity in .data.result; errored → .data.errorCode
result=$(echo "$poll" | jq '.data.result')
error_code=$(echo "$poll" | jq -r '.data.errorCode')

# Optional long poll when a human is waiting:
# poll=$(rc task "$task_id" --wait --timeout 300 --json)
# Or block on fetch itself (unwraps result into the entity path):
# rc fetch person --url https://linkedin.com/in/jane --live --wait --timeout 300 --json
```

Live can finish in under 15s or take tens of minutes. Trial plans cannot call live routes. On poll timeout, exit 4 with `poll_timeout`; re-run `rc task` later. On task failure, exit 1 with `task_errored`.
