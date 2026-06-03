---
name: datadog
description: Use the Datadog CLI (`pup`) for all Datadog observability work — monitors, logs, metrics, APM/traces, incidents, security, SLOs, dashboards, infrastructure, and many other API domains. ALWAYS prefer `pup` over web fetching, the Datadog UI, or the Datadog MCP server. Trigger when the user references Datadog, pastes a `*.datadoghq.com` / `app.datadoghq.eu` URL, names a monitor/SLO/dashboard/incident/service, asks to investigate errors / latency / alerts / security signals, or runs anything starting with `pup`. Reads and most writes are fine; destructive `delete` / `cancel` and bulk writes need explicit confirmation.
---

# Using the pup CLI

`pup` is the canonical interface to Datadog's APIs from a terminal, spanning the full observability platform. Prefer it over web fetches, the Datadog UI, the Datadog MCP server, or hand-rolled `curl` against the API — it handles auth, multi-site/multi-org routing, structured JSON output, and confirmation auto-approval for you.

## When to reach for pup

- The user mentions Datadog, or pastes a `datadoghq.com` / `datadoghq.eu` / `*.datadoghq.com` URL
- Investigating production errors, latency, alerts, or incidents — `pup logs`, `pup traces`, `pup monitors`, `pup incidents`
- Checking the health of a service or team — `pup slos`, `pup monitors`, `pup apm services`
- Querying metrics, searching logs, finding slow traces, or aggregating any of them
- Security signals/findings, infrastructure hosts, dashboards, synthetics, on-call, cost — there is almost certainly a `pup` command
- Run `pup agent schema --compact` for the live, authoritative list of command groups (the surface is broad — when in doubt, check)
- Anything starting with `pup`

If the answer might be in Datadog, try `pup` before web fetching or asking the user.

## Key principles

- **Agent mode is auto-detected here.** pi sets `PI_CODING_AGENT=1`, so `pup` runs in **agent mode**: `--help` returns a JSON schema (not text), errors are structured JSON with `suggestions`, confirmation prompts are auto-approved, and output is wrapped in a `{status, data, metadata}` envelope. This is what you want when consuming output programmatically.
- **Use `--no-agent` for any command you hand to the user.** Scripts, aliases, and runbooks run outside this session won't have an agent env var set, so `pup` emits the raw payload (no envelope). A script doing `pup ... | jq '.data[]'` breaks for the user. Use `pup --no-agent ... | jq '.[]'` in anything they'll run later.
- **Discover commands with `--help` and `pup agent schema`** rather than guessing flags — the schema is generated from the live command tree and stays in sync.
- **Prefer dedicated subcommands.** Each domain (`monitors`, `logs`, `metrics`, …) has its own verbs. There's no raw-API escape hatch you need for normal work.
- **APM/trace durations are in NANOSECONDS** — 1ms = 1,000,000; 1s = 1,000,000,000. This is the most common footgun.
- **Always pass `--from`, start narrow, filter at the API.** Begin with `1h` and widen only if needed; use `--tags` / `--query` / `--name` instead of fetching everything and filtering locally.

## Prerequisites

Assume `pup` is installed and authenticated. If a command fails with an auth error (401/403), tell the user to authenticate (e.g. `pup auth login`) and try again — don't run auth flows for them. Don't blindly retry auth failures.

If the user works across multiple Datadog orgs/sites, commands accept `--org <session>` to target a named session; otherwise the default is used. If they haven't specified one and the default works, just run the command.

## Discovery (recommended first steps)

In agent mode these return JSON schema, not prose. When unsure about a command's flags, check `pup <domain> --help` before guessing.

```bash
pup --help                    # full schema: commands, flags, query syntax, workflows, anti-patterns
pup logs --help               # domain subtree: only logs commands + logs query syntax
pup monitors --help

# Explicit, work regardless of agent mode:
pup agent schema              # full JSON schema
pup agent schema --compact    # names + flags only (fewer tokens)
```

## Output formats

```bash
pup monitors list --output=json    # default; recommended for agents
pup monitors list --output=table   # human-readable
pup monitors list --output=yaml
pup monitors list --output=csv
```

In agent mode the JSON is wrapped: `{ "status": "success", "data": [...], "metadata": { "count", "truncated", "command", "warnings" } }`. Errors: `{ "status": "error", "error_code", "error_message", "operation", "suggestions": [...] }`. Pipe with `jq '.data[]'` in-session; `pup --no-agent ... | jq '.[]'` for the user.

## Command reference

All commands follow `pup <domain> <action> [flags]` or `pup <domain> <subgroup> <action> [flags]`. CRUD shape: `list` / `get <id>` / `create --file f.json` / `update <id> --file f.json` / `delete <id> --yes`.

### Logs

```bash
pup logs search --query="status:error" --from=1h --limit=100
pup logs search --query="service:payment-api @http.status_code:5*" --from=24h
pup logs aggregate --query="*" --from=1h --compute="count" --group-by="service"
pup logs aggregate --query="service:api" --from=1h --compute="avg(@duration)" --group-by="service"
```

**Use `aggregate` for counts/stats** — never fetch raw logs to count them yourself.

### Metrics

```bash
pup metrics query --query="avg:system.cpu.user{env:prod} by {host}" --from=1h
pup metrics query --query="sum:trace.express.request.hits{service:api}" --from=1h
pup metrics list --filter="system.*"
```

### APM / Services / Traces

```bash
pup apm services list --env production
pup apm services stats --env production
pup apm services operations --env production --service my-service
pup apm dependencies list --env production

# Durations in NANOSECONDS — 1000000000 = 1s
pup traces search --query="service:api status:error" --from=1h
pup traces search --query="service:api @duration:>1000000000" --from=1h   # > 1s (1e9 ns)
pup traces aggregate --query="service:api" --compute="avg(@duration)" --group-by="resource_name" --from=1h
```

### Monitors

```bash
pup monitors list --tags="env:production" --name="CPU"
pup monitors search --query="status:Alert"
pup monitors get 12345678
pup monitors create --file monitor.json       # write — confirm first
pup monitors delete 12345678 --yes            # destructive — confirm first
```

### Incidents

```bash
pup incidents list
pup incidents list --query="status:active" --limit=20
pup incidents get <incident-id>
```

### Dashboards / SLOs / Synthetics / Downtimes

```bash
pup dashboards list
pup dashboards get abc-123
pup dashboards url abc-123 --from=now-1w --to=now --live=true   # note: anchored time syntax, see Time ranges

pup slos list
pup slos status slo-123 --from=30d --to=now

pup synthetics tests list
pup synthetics tests search --text="login"

pup downtime list
pup downtime cancel abc-123-def               # destructive — confirm first
```

### Security

```bash
pup security signals list --query="severity:critical" --from=24h
pup security rules list
pup security findings analyze ...             # DDSQL analytics for Cloud/App security
```

### Infrastructure / Events

```bash
pup infrastructure hosts list --filter="env:prod" --count=100
pup events list --tags="source:deploy" --from=24h
pup events search --query="deploy" --from=24h
```

### Live Debugger (runtime values without redeploying)

```bash
pup debugger context my-svc --env prod                        # verify active instances
pup symdb search --service my-svc --query MyController --view probe-locations
pup debugger probes create --service my-svc --env prod \
  --probe-location "com.example.MyController:handleRequest" \
  --capture "request.id" --ttl 1h                             # write — confirm first
pup debugger probes watch <PROBE_ID> --fields "message,captures,timestamp" --limit 10
pup debugger probes delete <PROBE_ID>                         # destructive — confirm first
```

Many more domains exist (RUM, cost, on-call, service-catalog, notebooks, obs-pipelines, llm-obs, users, integrations, cicd, workflows, runbooks, …). Run `pup --help` or `pup agent schema --compact` for the full list.

## Query syntax

```
status:error                     # filter by field
service:web-app                  # filter by service
@user.id:12345                   # custom attribute (@ prefix)
host:i-*                         # wildcard
"exact error message"            # exact phrase
status:error AND service:web     # boolean AND (implicit or explicit)
status:error OR status:warn      # boolean OR
-status:info                     # negation
@http.status_code:[400 TO 599]   # numeric range
```

Metrics: `<aggregation>:<metric>{<filter>} by {<group>}`, e.g. `avg:system.cpu.user{env:prod} by {host}`.

## Time ranges

Most time-series commands (`logs`, `traces`, `metrics`, `events`, `security`, …) take `--from` / `--to` as a duration-ago: relative (`1h`, `30m`, `7d`, `1w`, `5min`, `"2 hours"`), RFC3339 (`2024-01-01T00:00:00Z`), Unix ms (`1704067200000`), or `now`.

A few commands (notably `dashboards url`) instead use **anchored expressions** like `now-1w` / `now-1h` / `now`. When in doubt, run `pup <command> --help` — it shows the accepted format and default for that command's `--from`/`--to`.

## Common workflows

**Error investigation:**

```bash
pup logs aggregate --query="status:error" --from=1h --compute="count" --group-by="service"   # 1. counts by service
pup logs search --query="status:error AND service:<name>" --from=1h --limit=20                # 2. drill in
pup monitors list --tags="service:<name>"                                                     # 3. related monitors
pup events list --from=4h                                                                      # 4. recent deploys/events
```

**Performance investigation:**

```bash
pup metrics query --query="avg:trace.servlet.request.duration{service:<name>} by {resource_name}" --from=1h
pup traces search --query="service:<name> AND @duration:>5000000000" --from=1h   # >5s, nanoseconds
pup metrics query --query="avg:system.cpu.user{service:<name>} by {host}" --from=1h
```

## Error handling

| Status | Meaning | Action |
|--------|---------|--------|
| 401 | Auth failed | Tell the user to re-authenticate — don't retry |
| 403 | Insufficient permissions | Tell the user their credentials lack the required scope — don't retry |
| 404 | Resource not found | Check the resource ID |
| 429 | Rate limited | Back off, add delays between calls |
| 5xx | Server error | Retry after a delay; check https://status.datadoghq.com/ |

In agent mode, the error JSON's `suggestions` field tells you the remediation — read it.

## Anti-patterns

- Omitting `--from` on time-series queries — you'll get unexpected ranges.
- `--limit=1000` or `--from=30d` as a first step — start small, widen only if needed.
- Listing all monitors/logs without filters in a large org.
- Fetching raw logs to count them — use `pup logs aggregate --compute=count`.

## Destructive and write operations

Reads (`list`, `get`, `search`, `aggregate`, `query`, `status`, `watch`) are always safe. Creates/updates and especially destructive ops affect the user's live Datadog org. **Agent mode auto-approves `--yes` prompts, so there is no interactive guard** — you must confirm with the user yourself before running:

- `pup <resource> delete ...` (monitors, dashboards, SLOs, debugger probes, datasets, api-keys, app-keys, …)
- `pup downtime cancel`, `pup cases archive`, `pup data-deletion requests create/cancel`
- `pup <resource> create` / `update` that mutates config (monitors, dashboards, obs-pipelines, workflows, security rules, …)
- `pup workflows run`, `pup debugger probes create`, `pup runbooks run` (executes multi-step procedures)
- `pup auth logout` — clears the user's stored credentials (and shared DCR client creds for the default session)

Before any of these, state exactly what will change and on which org/site, and wait for explicit go-ahead. Before a batch of writes (e.g. creating many resources in a loop), show the full plan first. **If unsure whether a command mutates, run `pup <command> --help` and read it. Default to not running it.**

## Filing issues and finding docs

- Bugs / feature requests for pup, its skills, agents, or docs all go to `DataDog/pup` — use the `github` skill (or ask the user).
- For Datadog product documentation, prefer the `docs` skill or `https://docs.datadoghq.com/llms.txt` (append `.md` to most doc URLs for raw markdown). For CLI-specific guidance, use `pup <domain> --help` or `pup agent schema`.
