# `DoJiraStuff` — Conversational Jira via Claude Code

> **Talk to your Jira board the same way you'd talk to a teammate.**
> No MCP server. No middleware. No GUI. Just Claude Code + the Jira REST API.

---

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   you  ──▶  claude code  ──▶  jira rest api v3  ──▶  jira     │
│                  ▲                                              │
│                  │                                              │
│             CLAUDE.md                                           │
│          (api playbook)                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## What This Is

`DoJiraStuff` is a **zero-dependency, conversational Jira interface** built on Claude Code. Instead of clicking through a UI or writing scripts, you describe what you want in plain English and Claude handles the Jira REST API calls, parses the responses, and presents results in a clean, readable format.

**The secret ingredient** is `CLAUDE.md` — a machine-readable playbook that lives alongside your code and teaches Claude exactly how to authenticate, which endpoints to hit, how to handle Atlassian Document Format (ADF) payloads, and what the gotchas are.

### Why not MCP?

MCP (Model Context Protocol) is powerful, but it introduces a server process, a config file, and an external dependency. This repo intentionally avoids all of that. Everything runs locally using tools you already have: `curl`, `python3`, and Claude Code.

---

## How It Works

```
┌──────────────────────────────────────────────────────────────────────┐
│  CONVERSATION                                                        │
│                                                                      │
│  You:    "Create a spike in PROJ about evaluating VAPI, 5 points"   │
│                                                                      │
│  Claude: 1. Reads .env for credentials                              │
│          2. Checks field metadata for PROJ/Spike issue type         │
│          3. Builds ADF-formatted payload                             │
│          4. POSTs to /rest/api/3/issue                              │
│          5. Returns: ✓ PROJ-123 created → [link]                    │
└──────────────────────────────────────────────────────────────────────┘
```

Claude maintains context across the conversation. You can refer back to tickets by key, chain operations ("now assign that to me and move it to In Progress"), and ask follow-up questions about results — all without restating context.

---

## Prerequisites

| Requirement | Notes |
|-------------|-------|
| [Claude Code](https://claude.ai/code) | `npm install -g @anthropic-ai/claude-code` |
| `curl` | Pre-installed on macOS |
| `python3` | Pre-installed on macOS — used for JSON parsing |
| Atlassian Cloud account | Must have project access |
| Jira API token | See setup below |

---

## Setup

### Step 1 — Get an Atlassian API Token

1. Visit **https://id.atlassian.com/manage-profile/security/api-tokens**
2. Click **Create API token**
3. Name it (e.g. `claude-code`)
4. **Copy the token immediately** — you won't see it again

### Step 2 — Configure your `.env`

```bash
cp .env.example .env
```

Edit `.env` with your values:

```env
JIRA_BASE_URL=https://your-org.atlassian.net
JIRA_EMAIL=you@yourcompany.com
JIRA_API_TOKEN=your_api_token_here
JIRA_DEFAULT_PROJECT=YOUR_PROJECT
```

> **`JIRA_DEFAULT_PROJECT`** is a fallback only. You can specify any project key inline in your requests — e.g. _"show me open ENG tickets"_ — and Claude will use that instead.

`.env` is gitignored and will never be committed.

### Step 3 — Verify your credentials

```bash
set -a && source .env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/myself" | python3 -m json.tool
```

You should see your Atlassian account details as JSON. If you get a 401, your token is wrong or expired.

### Step 4 — Open in Claude Code

```bash
cd /path/to/DoJiraStuff
claude
```

Claude will automatically read `CLAUDE.md` for its operating instructions. Start talking.

---

## Project Structure

```
DoJiraStuff/
│
├── CLAUDE.md          ← The brains: API patterns, curl templates,
│                        field reference, ADF format, gotchas.
│                        Claude reads this before every operation.
│
├── README.md          ← You are here
│
├── .env               ← Your credentials (gitignored, never committed)
├── .env.example       ← Safe template — edit and copy to .env
└── .gitignore         ← Protects .env from accidental commits
```

### The Role of `CLAUDE.md`

`CLAUDE.md` is the core of this project. It is not documentation for humans — it is **operating instructions for Claude**. It covers:

- How to source credentials with `set -a`
- How to resolve the project key from your request vs. the default
- `curl` templates for every CRUD operation
- Jira's Atlassian Document Format (ADF) for rich text fields
- How to discover required fields and allowed values for any project
- How to fetch and apply workflow transitions
- Response formatting guidelines

If you add a new project with different required fields, update `CLAUDE.md` and Claude will know about it in the next session.

---

## What You Can Do

### Read

```
Show me all open tickets in PROJ
What's the status of PROJ-42?
List all bugs assigned to me
Show me high priority issues in the current sprint
Search for tickets mentioning "authentication" in ENG
Give me everything that's In Progress across PROJ
```

### Create

```
Create a spike in PROJ: evaluate VAPI as a dialer provider, 5 points
File a bug in ENG: login crashes on Safari, high priority
Create a story: "As a user I want to reset my password"
Add a task to PROJ about updating the API docs
```

### Update

```
Move PROJ-42 to In Progress
Assign ENG-15 to me
Set PROJ-7 priority to High
Update the summary of PROJ-42 to "Fix Safari login crash"
Add acceptance criteria to PROJ-123
```

### Comment

```
Add a comment to PROJ-42: "Fixed in PR #123, deploying Friday"
Comment on ENG-15 that it's blocked by PROJ-9
```

### Delete

```
Delete PROJ-42
```

---

## Multi-Project Support

Claude resolves the project key from your request directly. There is no single locked-in project.

| Request | Project used |
|---------|-------------|
| `"show me open PROJ tickets"` | `PROJ` |
| `"create a bug in ENG"` | `ENG` |
| `"show me my tickets"` | `$JIRA_DEFAULT_PROJECT` |
| `"what's the status of OPS-7?"` | `OPS` |

The `JIRA_DEFAULT_PROJECT` in `.env` is only a fallback for ambiguous requests.

---

## Supported Issue Types

Issue types vary by Jira project configuration. Common types include:

| Type | Use for |
|------|---------|
| `Story` | User-facing features |
| `Task` | Internal work items |
| `Spike` | Research and investigation |
| `Bug` | Defects found in testing |
| `Epic` | Large bodies of work |
| `Sub-task` | Breakdown of a parent issue |

To see what issue types are available in your project:

```bash
set -a && source .env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/issue/createmeta?projectKeys=YOUR_PROJECT" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
for p in data['projects']:
    for it in p['issuetypes']:
        print(it['id'], '-', it['name'])
"
```

---

## Atlassian Document Format (ADF)

Jira's REST API v3 **does not accept plain strings** for rich text fields. All description, comment, and acceptance criteria values must use ADF — a JSON document structure.

The minimal ADF wrapper:

```json
{
  "type": "doc",
  "version": 1,
  "content": [
    {
      "type": "paragraph",
      "content": [
        { "type": "text", "text": "Your text here." }
      ]
    }
  ]
}
```

Claude handles this automatically. You never need to write ADF yourself.

---

## Discovering Required Fields

Every Jira project can have different required fields and custom field IDs. Before creating issues in a new project, Claude will fetch the field metadata automatically. You can also do it manually:

```bash
set -a && source .env && set +a
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/issue/createmeta?projectKeys=YOUR_PROJECT&expand=projects.issuetypes.fields" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
for p in data['projects']:
    for it in p['issuetypes']:
        print('\\nIssue type:', it['name'])
        for k, v in it['fields'].items():
            req = ' (REQUIRED)' if v.get('required') else ''
            vals = [str(x.get('id')) + ':' + str(x.get('value','')) for x in v.get('allowedValues', [])]
            print(' ', k, '-', v.get('name','') + req, '|', ', '.join(vals) if vals else '')
"
```

---

## Reproducibility

To set this up on a new machine from scratch:

```bash
git clone https://github.com/YOUR_ORG/DoJiraStuff
cd DoJiraStuff
cp .env.example .env
# edit .env and add your API token
claude
```

That's it. No `npm install`. No `pip install`. No build step. The only runtime dependencies are `curl` and `python3`, both pre-installed on macOS.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `401 Unauthorized` | Bad or expired API token | Regenerate at https://id.atlassian.com/manage-profile/security/api-tokens |
| `URL rejected: No host part` | `.env` not loaded | Use `set -a && source .env && set +a`, not just `source .env` |
| `400 Bad Request` on create | ADF format wrong | Claude knows the format — retry with more specific instructions |
| `"Field X is required"` | Missing required field | Ask Claude to fetch field metadata first, then retry |
| Transition fails | Wrong transition ID | Claude fetches available transitions first before applying |
| Field missing from results | Not requested | Ask Claude to include specific fields |

---

## Security

- Your API token lives **only** in `.env` on your local machine
- `.gitignore` ensures `.env` is never committed
- The token carries the same permissions as your Jira account — treat it like a password
- To revoke: https://id.atlassian.com/manage-profile/security/api-tokens
- **Never paste your token into a prompt, issue, or comment** — it will appear in history

---

## Configuration Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `JIRA_BASE_URL` | Yes | Your Atlassian Cloud URL, e.g. `https://your-org.atlassian.net` |
| `JIRA_EMAIL` | Yes | Email associated with your Atlassian account |
| `JIRA_API_TOKEN` | Yes | API token from id.atlassian.com |
| `JIRA_DEFAULT_PROJECT` | Yes | Fallback project key for ambiguous requests |

---

*Built with [Claude Code](https://claude.ai/code) — powered by the Jira REST API v3*
