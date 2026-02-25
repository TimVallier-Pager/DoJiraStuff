# Claude Code — Jira Assistant

This project lets you manage Jira issues conversationally. Claude reads your credentials from `.env` and makes direct calls to the Jira REST API v3 using `curl`.

> If a `CLAUDE.local.md` file exists in this directory, read it — it contains project-specific configuration, cached field IDs, and custom workflows.

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
- `project = PROJECT AND text ~ "keyword"` — full-text search
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

Common issue types: `Task`, `Bug`, `Story`, `Epic`, `Spike`, `Sub-task`
Priority values: `Highest`, `High`, `Medium`, `Low`, `Lowest`

To set priority: `"priority": { "name": "High" }`
To set assignee: `"assignee": { "accountId": "ACCOUNT_ID_HERE" }`
To set sprint: `"customfield_10010": SPRINT_ID` (bare integer)

### Update an issue

```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -X PUT \
  "$JIRA_BASE_URL/rest/api/3/issue/PROJECT-42" \
  -d '{"fields": {"summary": "Updated summary"}}'
```

### Transition an issue (change status)

First, get available transitions:
```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/issue/PROJECT-42/transitions" | python3 -c "
import json, sys
data = json.load(sys.stdin)
for t in data.get('transitions', []):
    print(t['id'], '-', t['name'])
"
```

Then apply:
```bash
curl -s -o /dev/null -w "%{http_code}" -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" -H "Content-Type: application/json" \
  -X POST "$JIRA_BASE_URL/rest/api/3/issue/PROJECT-42/transitions" \
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
curl -s -o /dev/null -w "%{http_code}" -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -X DELETE "$JIRA_BASE_URL/rest/api/3/issue/PROJECT-42"
```

### Link two issues

```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
# Get available link types first
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/issueLinkType" | python3 -c "
import json, sys
for lt in json.load(sys.stdin).get('issueLinkTypes', []):
    print(lt['id'], '-', lt['name'])
"
# Then create the link
curl -s -o /dev/null -w "%{http_code}" -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" -H "Content-Type: application/json" \
  -X POST "$JIRA_BASE_URL/rest/api/3/issueLink" \
  -d '{"type":{"id":"LINK_TYPE_ID"},"inwardIssue":{"key":"PROJ-1"},"outwardIssue":{"key":"PROJ-2"}}'
```

### Look up a user by email

```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/user/search?query=someone@yourcompany.com"
```

---

## Response Formatting

When displaying Jira results to the user:
- Show issue key (e.g. `PROJ-42`) as a bold label
- Include summary, status, assignee, and priority when available
- For lists, use a compact table or bulleted list
- For a single issue, show full details including description
- Always include the direct URL: `$JIRA_BASE_URL/browse/ISSUE-KEY`
- Use `-o /dev/null -w "%{http_code}"` on write operations for clean output

---

## Discovering Required Fields

Before creating an issue in an unfamiliar project, fetch field metadata:

```bash
set -a && source /path/to/DoJiraStuff/.env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H "Accept: application/json" \
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
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H "Accept: application/json" \
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

## Notes

- **ADF required** — Jira REST API v3 rejects plain strings for rich text fields (description, comment body, acceptance criteria). Always use the ADF doc structure shown above.
- **URL-encode JQL** — spaces → `%20`, `=` → `%3D`, `"` → `%22`
- **204 = success** — PUT/DELETE return no body on success
- **Sprint field** — pass as bare integer (`customfield_10010: 6943`), not `{ "id": N }`
- **Select fields** — pass as `{ "id": "OPTION_ID" }`, not the display name string
- **`set -a` required** — plain `source .env` doesn't export vars to subshells; always use `set -a && source /path/.env && set +a`
