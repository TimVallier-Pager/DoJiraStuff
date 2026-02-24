# Claude Code — Jira Assistant

This project lets you manage Jira issues conversationally. Claude reads your credentials from `.env` and makes direct calls to the Jira REST API v3 using `curl`.

---

## Credentials

Always load credentials from `.env` before making any API call. Use `set -a` to auto-export all variables:

```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
```

Use the absolute path to `.env` to ensure it loads correctly regardless of working directory.

Variables available after sourcing:
- `JIRA_BASE_URL` — your Atlassian Cloud base URL
- `JIRA_EMAIL` — your Atlassian account email
- `JIRA_API_TOKEN` — the API token
- `JIRA_DEFAULT_PROJECT` — fallback project key when none is specified in the request

## Project Key Resolution

Always determine the project key from the user's request first. Fall back to `JIRA_DEFAULT_PROJECT` only if no project is mentioned.

Examples:
- "show me open PROJ tickets" → use `PROJ`
- "create a bug in ENG" → use `ENG`
- "show me open tickets" → use `$JIRA_DEFAULT_PROJECT`

Use the resolved key everywhere a project key is needed in API calls.

---

## Auth Pattern

All requests use HTTP Basic Auth: `email:api_token` passed via `-u` flag.

```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/myself"
```

---

## Common Operations

### Get a specific issue

Replace `PROJECT` with the resolved project key (e.g. `PROJ`, `ENG`).

```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/issue/PROJECT-42"
```

### Search issues (JQL)

```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/search?jql=project%3DPROJECT%20AND%20status%3D%22In%20Progress%22&fields=summary,status,assignee,priority"
```

Useful JQL patterns (substitute the actual project key for `PROJECT`):
- `project = PROJECT AND status = "To Do"` — backlog
- `project = PROJECT AND assignee = currentUser()` — my issues
- `project = PROJECT AND status != Done ORDER BY created DESC` — open issues
- `project = PROJECT AND priority = High` — high priority
- `project = PROJECT AND text ~ "login"` — full-text search
- `project = PROJECT AND sprint in openSprints()` — current sprint

### Create an issue

```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -X POST \
  "$JIRA_BASE_URL/rest/api/3/issue" \
  -d '{
    "fields": {
      "project": { "key": "PROJECT" },
      "summary": "Issue summary here",
      "description": {
        "type": "doc",
        "version": 1,
        "content": [
          {
            "type": "paragraph",
            "content": [{ "type": "text", "text": "Description text here." }]
          }
        ]
      },
      "issuetype": { "name": "Task" }
    }
  }'
```

Issue types available: `Task`, `Bug`, `Story`, `Epic`, `Subtask`
Priority values: `Highest`, `High`, `Medium`, `Low`, `Lowest`

To set priority, add to `fields`:
```json
"priority": { "name": "High" }
```

To set assignee (use accountId):
```json
"assignee": { "accountId": "ACCOUNT_ID_HERE" }
```

### Update an issue

```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -X PUT \
  "$JIRA_BASE_URL/rest/api/3/issue/PROJECT-42" \
  -d '{
    "fields": {
      "summary": "Updated summary"
    }
  }'
```

### Transition an issue (change status)

First, get available transitions:
```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/issue/PROJECT-42/transitions"
```

Then apply the transition using its `id`:
```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -X POST \
  "$JIRA_BASE_URL/rest/api/3/issue/PROJECT-42/transitions" \
  -d '{"transition": {"id": "TRANSITION_ID"}}'
```

### Add a comment

```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -X POST \
  "$JIRA_BASE_URL/rest/api/3/issue/PROJECT-42/comment" \
  -d '{
    "body": {
      "type": "doc",
      "version": 1,
      "content": [
        {
          "type": "paragraph",
          "content": [{ "type": "text", "text": "Your comment here." }]
        }
      ]
    }
  }'
```

### Delete an issue

```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -X DELETE \
  "$JIRA_BASE_URL/rest/api/3/issue/PROJECT-42"
```

### Get project metadata (issue types, fields, etc.)

```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/project/PROJECT"
```

### Look up a user by email (to get accountId for assignment)

```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/user/search?query=someone@yourcompany.com"
```

---

## Response Formatting

When displaying Jira results to the user:
- Show issue key (e.g. `PROJ-42`, `ENG-7`) as a bold label
- Include summary, status, assignee, and priority when available
- For lists of issues, use a compact table or bulleted list
- For a single issue, show full details including description
- Always include the direct URL: `$JIRA_BASE_URL/browse/ISSUE-KEY`

---

## Discovering Required Fields

Before creating an issue in an unfamiliar project, fetch the field metadata to find required fields and allowed values:

```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/issue/createmeta?projectKeys=PROJECT&issuetypeIds=ISSUETYPE_ID&expand=projects.issuetypes.fields" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
for p in data['projects']:
    for it in p['issuetypes']:
        for k, v in it['fields'].items():
            req = '(required)' if v.get('required') else ''
            vals = [str(x.get('id')) + ':' + str(x.get('value','')) for x in v.get('allowedValues', [])]
            print(k, '-', v.get('name',''), req, '|', ', '.join(vals) if vals else '')
"
```

To get issue type IDs for a project:
```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/issue/createmeta?projectKeys=PROJECT" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
for p in data['projects']:
    for it in p['issuetypes']:
        print(it['id'], '-', it['name'])
"
```

---

## Release Ticket Process (RAG project)

When asked to "make a release ticket" for one or more tickets, follow these steps in order:

### 1. Find the next sprint

RAG board ID is **210** — no lookup needed. Fetch sprints directly:

```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/agile/1.0/board/210/sprint?state=active,future" | python3 -c "
import json, sys
data = json.load(sys.stdin)
for s in data.get('values', []):
    print(s['id'], '-', s['name'], '-', s['state'])
"
```

The current sprint has `state: active`. The next sprint is the first one with `state: future`.

### 2. Create the release ticket

- **Project:** RAG
- **Issue type:** Task
- **Title:** `Release RAG-XXXX RAG-XXXX` (list all bundled ticket keys)
- **Story Points:** 1
- **Product Type:** Features (`customfield_10290: { "id": "10475" }`)
- **Sprint:** next sprint ID (`customfield_10010: SPRINT_ID` — pass as a bare integer, **not** `{ "id": N }`)
- **Description (ADF):** "The goal of this ticket is to release the following ticket(s):" followed by a bullet list of each ticket key and summary
- **Acceptance Criteria (ADF):** "Release the aforementioned tickets to production without regression."

### 3. Link the release ticket to each original ticket

"Implements" link type ID is **10201** — no lookup needed. Create a link for each original ticket:
```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -X POST \
  "$JIRA_BASE_URL/rest/api/3/issueLink" \
  -d '{
    "type": { "name": "Implements" },
    "inwardIssue": { "key": "ORIGINAL-KEY" },
    "outwardIssue": { "key": "RELEASE-KEY" }
  }'
```

### 4. Comment on each original ticket

Leave a natural, conversational comment on each original ticket noting that it has completed acceptance testing and will be released in the linked release ticket. Include the release ticket key and a link. Use ADF format:

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -X POST \
  "$JIRA_BASE_URL/rest/api/3/issue/ORIGINAL-KEY/comment" \
  -d '{
    "body": {
      "type": "doc",
      "version": 1,
      "content": [
        {
          "type": "paragraph",
          "content": [{ "type": "text", "text": "YOUR NATURAL COMMENT HERE" }]
        }
      ]
    }
  }'
```

### 5. Transition each original ticket to Done (Resolved)

Done (Resolved) transition ID is **131** — no lookup needed.

```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -X POST \
  "$JIRA_BASE_URL/rest/api/3/issue/ORIGINAL-KEY/transitions" \
  -d '{"transition": {"id": "131"}}'
```

### Summary checklist

- [ ] Found next sprint ID
- [ ] Created release ticket (Task, 1pt, Features, next sprint, correct title/body/AC)
- [ ] Linked each original ticket to release ticket as "implements"
- [ ] Left a comment on each original ticket referencing the release ticket
- [ ] Transitioned each original ticket to Done (Resolved)

---

## Notes

- The Jira REST API v3 uses Atlassian Document Format (ADF) for rich text fields (description, comment body, acceptance criteria). Always use the ADF structure shown above — plain strings will be rejected.
- URL-encode JQL when passing as a query param (spaces → `%20`, `=` → `%3D`, `"` → `%22`), or use `--data-urlencode` with curl's `-G` flag.
- A 204 response on PUT/DELETE means success (no body returned).
- Transitions require fetching available transition IDs first — they vary per project workflow.
- Always use `set -a && source /path/to/.env && set +a` (not just `source .env`) to ensure variables are exported to subshells.
- Select/option custom fields require passing `{ "id": "OPTION_ID" }` — not the display name string.
