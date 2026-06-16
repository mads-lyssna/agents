---
name: notion
description: Use the Notion CLI (`ntn`) for Notion page, database/data source, file upload, and authenticated Notion API work. ALWAYS prefer `ntn` over web fetching, the Notion UI, or raw `curl` against the Notion API. Trigger when the user explicitly references Notion, pastes a `notion.so` / `notion.site` / `notion.com` URL, provides a Notion page/database/data-source ID, asks to read/query/create/update Notion content, or runs anything starting with `ntn`. Reads are fine. Small, explicit page writes are fine after confirming the target and effect; page body replacement, trash/archive actions, file uploads, auth changes, and bulk writes need explicit confirmation. Do not manage Notion Workers unless the user explicitly asks.
---

# Using the ntn CLI for Notion pages

`ntn` is the canonical terminal interface for page-focused Notion work. Prefer it over web fetches, the Notion UI, or hand-rolled `curl` because it handles authentication, workspace targeting, Notion API versions, Markdown page conversion, and authenticated API requests.

## When to use this skill

Use `ntn` when the user provides an explicit Notion signal:

- They mention Notion directly
- They paste a `notion.so`, `notion.site`, or `notion.com` URL
- They provide a Notion page, database, data source, block, comment, user, or file upload ID
- They ask to read, create, update, trash, search, or query Notion content
- They ask to query a Notion database/data source or resolve a database ID
- They ask to upload or inspect Notion files
- They ask to make an authenticated Notion API request
- They ask to check CLI setup/auth/network health with `ntn doctor`
- They run or ask about a command starting with `ntn`

Do not use this skill just because the user says “page” or “database” generically. If the user indicates the target content is in Notion, use `ntn` before web fetching or asking them to use the UI.

## Key principles

- **Assume `ntn` is installed and authenticated.** If auth fails, report the failure and suggest `ntn login` or `ntn doctor`; do not run login flows unless explicitly asked.
- **Target the right workspace.** Use `NOTION_WORKSPACE_ID` when workspace selection matters. If a command prompts for a workspace, ask the user which workspace to use instead of guessing.
- **Prefer dedicated subcommands.** Use `ntn pages`, `ntn datasources`, and `ntn files` before falling back to `ntn api`.
- **Use JSON for programmatic output.** Prefer `--json` when supported.
- **Use stdin or temp files for Markdown/JSON bodies.** Avoid complex inline shell quoting for multi-line page content, data-source filters, or API payloads.
- **Inspect before mutating.** Read the current page or query the current entries before creating, replacing, archiving, trashing, or otherwise mutating Notion state.
- **Check path-specific help before guessing.** Use `ntn <subcommand> --help` and `ntn api <path> --help` for flags and endpoint details.

## Diagnostics and environment

```bash
ntn --version
ntn doctor
```

Useful global flags and environment variables:

```bash
ntn <command> --verbose

export NOTION_WORKSPACE_ID=<workspace-id>       # skip workspace prompt
export NOTION_API_TOKEN=<token>                 # override stored keychain auth
export NOTION_KEYRING=0                         # use ~/.config/notion/auth.json instead of OS keychain
export NOTION_API_VERSION=<version>             # pin a Notion API version when required
```

Do not change auth state (`ntn login`, `ntn logout`) unless the user explicitly asks. If auth is broken, report the failure and suggest `ntn login` / `ntn doctor`.

## Page IDs and Notion URLs

If the user provides a Notion URL, extract the page or database ID from the URL. Notion IDs are 32 hex characters and may be displayed with hyphens. Preserve the ID exactly if possible.

Common URL patterns include IDs embedded after the title slug or query parameters. If the target ID is ambiguous, ask the user which page/database to use rather than guessing.

## Reading pages

Pages are returned as Markdown by default. Use `--json` when you need structured page metadata/content.

```bash
ntn pages get <page-id>
ntn pages get <page-id> --json
```

Use Markdown output for human-readable content review and JSON output when you need page metadata, properties, or structured data for scripts.

## Creating pages from Markdown

Create pages with Markdown content and an explicit parent reference:

```bash
ntn pages create --parent page:<page-id> --content "# New child page"
ntn pages create --parent database:<database-id> --content "# New database page"
ntn pages create --parent data-source:<data-source-id> --content "# New data source entry"
```

For multi-line Markdown, write a temp file and pipe it through stdin:

```bash
body=$(mktemp -t notion-page.XXXXXX.md)
cat > "$body" <<'MARKDOWN'
# Project update

## Summary

Shipped the first milestone.
MARKDOWN

ntn pages create --parent page:<parent-page-id> < "$body"
```

Confirm first unless the user explicitly asked you to create the page. Before creating database/data-source entries, confirm the intended parent and required properties.

## Updating page content safely

`ntn pages update <page-id>` replaces the page body with the Markdown you provide. Treat it as a full-body replacement, not a targeted text edit. It may not preserve every Notion block, embedded object, or unsupported structure after Markdown conversion.

Default safe workflow:

1. Read the existing page:

   ```bash
   ntn pages get <page-id> > current.md
   ```

2. Prepare the full replacement Markdown in a temp file.
3. Show or summarize the replacement/diff and the target page.
4. Ask for confirmation unless the user explicitly requested the overwrite.
5. Update from stdin:

   ```bash
   ntn pages update <page-id> < replacement.md
   ```

Use `--content` only for short one-line replacements:

```bash
ntn pages update <page-id> --content "# Replacement content"
```

Important page safety rules:

- Do not use `pages update` for a small edit unless you intend to replace the entire page body.
- Prefer `ntn api -X PATCH` for page property/metadata changes instead of replacing the page body.
- `--allow-deleting-content` permits deletion of child pages and databases during an update. Treat it as destructive and confirm first.
- `pages trash` moves a page to trash. Confirm first, even when using `--yes`.

```bash
ntn pages trash <page-id> --yes
```

## Updating page properties

Use `ntn api` when changing page properties or other fields that are not page body Markdown.

Examples:

```bash
ntn api "v1/pages/$PAGE_ID" -X PATCH archived:=false

ntn api "v1/pages/$PAGE_ID" -X PATCH \
  properties[Priority][number]:=2

ntn api "v1/pages/$PAGE_ID" -X PATCH \
  properties[Build version][rich_text][0][text][content]="2026.05.11"
```

Confirm before any property update unless the user explicitly asked for the exact change. Use bracket notation for property names with spaces or punctuation.

## Searching and inspecting content with the API

Use `ntn api` when no dedicated command covers the endpoint.

```bash
ntn api ls
ntn api ls --json
ntn api /v1/users/me
ntn api /v1/pages/<page-id>
ntn api v1/search query=roadmap
ntn api v1/search filter:='{"property":"object","value":"page"}' page_size:=10
```

Inspect endpoint syntax before constructing unfamiliar requests:

```bash
ntn api v1/comments --help
ntn api v1/comments --spec -X POST
ntn api v1/comments --docs -X POST
```

If a path supports multiple methods, pass `-X` with `--help`, `--spec`, or `--docs` so the CLI knows which operation to inspect.

## `ntn api` request syntax

`ntn api` adds `Authorization` and `Notion-Version` headers automatically. It uses CLI authentication by default, or `NOTION_API_TOKEN` when set.

Without request body input, `ntn api` sends `GET`:

```bash
ntn api v1/pages/$PAGE_ID
ntn api /v1/pages/$PAGE_ID
```

When body fields, `--data`, or stdin JSON are present, `ntn api` sends `POST` unless you override the method:

```bash
ntn api "v1/pages/$PAGE_ID" -X PATCH archived:=true
```

Inline request forms:

| Form | Meaning | Example |
| --- | --- | --- |
| `path=value` | Body field with a string value | `parent[page_id]=abc123` |
| `path:=json` | Body field parsed as JSON | `archived:=true` |
| `name==value` | Query parameter | `page_size==100` |
| `Header:Value` | Request header | `Accept:application/json` |

Use `=` for strings and `:=` for booleans, numbers, arrays, objects, strings that should be JSON-parsed, or `null`:

```bash
ntn api v1/search query=roadmap
ntn api v1/search page_size:=10
ntn api v1/search filter:='{"property":"object","value":"page"}'
```

For nested objects, use bracket or dot notation. Bracket notation is safest for Notion property names with spaces or punctuation:

```bash
ntn api v1/pages \
  parent[page_id]="$PARENT_PAGE_ID" \
  properties[Name][title][0][text][content]="CLI-created page"
```

Use explicit array indexes when order matters:

```bash
ntn api "v1/blocks/$PAGE_ID/children" -X PATCH \
  children[0][type]=paragraph \
  children[0][paragraph][rich_text][0][text][content]="First paragraph" \
  children[1][type]=heading_2 \
  children[1][heading_2][rich_text][0][text][content]="Next section"
```

Use `[]` to append repeated values in input order:

```bash
ntn api v1/comments \
  parent[page_id]="$PAGE_ID" \
  rich_text[][text][content]="First comment line" \
  rich_text[][text][content]="Second comment line"
```

For complex JSON bodies, use stdin or `--data` rather than inline shell arguments:

```bash
jq -n --arg page_id "$PARENT_PAGE_ID" '{
  parent: { page_id: $page_id },
  properties: {
    title: {
      title: [{ text: { content: "Generated page" } }]
    }
  }
}' | ntn api v1/pages

ntn api v1/search --data '{"query":"roadmap","page_size":10}'
```

Only use one body source per request: stdin JSON, `--data`, or inline body fields. You can combine headers and query parameters with any one body source.

Use `--notion-version` for one request only when a specific API version is required:

```bash
ntn api v1/users/me --notion-version <version>
```

Use `--verbose` to debug request method, URL, headers, body, response status, response headers, and response body:

```bash
ntn --verbose api v1/pages/$PAGE_ID
```

Do not use `--unsafe-verbose` unless the user explicitly asks and understands it may expose bearer tokens.

## Data sources and databases

Notion's current API uses data sources for database entries. If the user gives a database ID and a data-source command needs a data source ID, resolve it first.

```bash
ntn datasources resolve <database-id>

ntn datasources query <data-source-id> --limit 25
ntn datasources query <data-source-id> --limit 100 --start-cursor <cursor>
ntn datasources query <data-source-id> --sort "Name asc" --sort "Created desc"
ntn datasources query <data-source-id> --filter '{"property":"Status","status":{"equals":"Done"}}'
ntn datasources query <data-source-id> --filter-file ./filter.json
```

For non-trivial filters, use a file:

```bash
filter=$(mktemp -t notion-filter.XXXXXX.json)
cat > "$filter" <<'JSON'
{
  "and": [
    { "property": "Status", "status": { "equals": "In progress" } },
    { "property": "Priority", "select": { "equals": "High" } }
  ]
}
JSON

ntn datasources query <data-source-id> --filter-file "$filter" --limit 50
```

Pagination:

- Start with a small `--limit`.
- If the response includes a next cursor, pass it as `--start-cursor <cursor>`.
- Avoid fetching an entire large workspace/database unless the user explicitly needs it.

## Files

```bash
ntn files create
ntn files list
ntn files get <upload-id>
```

Use `ntn files create --help` before uploading; file upload flags may depend on CLI version. Uploads create Notion-hosted file objects, so confirm if the user has not clearly asked to upload.

## Notion Workers

This skill is page-focused. Do not manage Notion Workers unless the user explicitly asks to work with Workers.

If the user explicitly asks for Workers work, inspect live help before acting:

```bash
ntn workers --help
ntn workers <subcommand> --help
```

Worker deploys, deletes, environment variable changes, OAuth token retrieval, sync triggers, sync state changes, and capability executions may affect live systems or expose secrets. State the intended action, target workspace/worker, and expected side effects, then wait for explicit confirmation before running them.

## Common workflows

### Read a Notion page

```bash
ntn pages get <page-id>
```

If the user provides a Notion URL, extract the page ID from the URL. If multiple IDs are present or the target is ambiguous, ask which page to use.

### Create a child page

```bash
body=$(mktemp -t notion-page.XXXXXX.md)
cat > "$body" <<'MARKDOWN'
# Project update

## Summary

Shipped the first milestone.
MARKDOWN

ntn pages create --parent page:<parent-page-id> < "$body"
```

Confirm first unless the user explicitly asked you to create the page.

### Replace a page body

```bash
ntn pages get <page-id> > current.md
# prepare replacement.md
ntn pages update <page-id> < replacement.md
```

Confirm after reviewing the target and replacement, unless the user explicitly requested the overwrite.

### Update a page property

```bash
ntn api "v1/pages/$PAGE_ID" -X PATCH \
  properties[Status][status][name]="Done"
```

Use `ntn api v1/pages/$PAGE_ID --docs -X PATCH` or `--spec -X PATCH` if the property payload shape is unclear.

### Search for pages

```bash
ntn api v1/search \
  query=roadmap \
  filter:='{"property":"object","value":"page"}' \
  page_size:=10
```

### Query a database from a URL

1. Extract the database ID from the Notion URL.
2. Resolve data sources:

   ```bash
   ntn datasources resolve <database-id>
   ```

3. Query the data source:

   ```bash
   ntn datasources query <data-source-id> --limit 25
   ```

## Confirm before writes and destructive operations

Reads are safe: `pages get`, `datasources query`, `datasources resolve`, `files list`, `files get`, `api` GET-style reads, endpoint inspection (`api ls`, `--help`, `--spec`, `--docs`), and `doctor`.

Confirm with the user before commands that mutate Notion, local files, auth state, secrets, or external systems, especially:

- `ntn pages create`, unless the user explicitly asked to create the exact page
- `ntn pages update`, because it replaces page body content
- `ntn pages update --allow-deleting-content`
- `ntn pages trash`
- `ntn files create`
- `ntn api` requests that create/update/archive/delete, including `POST`, `PATCH`, and `DELETE`
- Bulk data-source/page writes through `ntn api`
- `ntn login`, `ntn logout`, `ntn update`
- Any Notion Workers command with live side effects or secret exposure

Before a write, state exactly what will change, the target workspace/page/database/data source when known, and whether local files or remote Notion state will be affected. For bulk writes, show the full plan and wait for explicit go-ahead.

## Common mistakes

- Triggering this skill for generic “page” or “database” language without an explicit Notion signal.
- Using a database ID where a data source ID is required — run `ntn datasources resolve <database-id>` first.
- Treating `pages update` as a targeted edit — it replaces the page body.
- Replacing a page body without reading the current content first.
- Using `pages update --allow-deleting-content` without understanding it can remove child pages/databases.
- Embedding multi-line Markdown/JSON in shell arguments — use stdin or temp files.
- Forgetting that body input makes `ntn api` default to `POST`; use `-X` for `GET`, `PATCH`, or another method.
- Using `=` for booleans/numbers/null in `ntn api`; use `:=` for JSON-typed values.
- Mixing multiple body sources in one `ntn api` call; choose stdin JSON, `--data`, or inline body fields.
- Falling back to raw API calls when a dedicated `ntn` command exists.
- Running Notion Workers commands when the user only asked about pages.
