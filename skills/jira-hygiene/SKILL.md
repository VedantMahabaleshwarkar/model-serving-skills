---
name: jira-hygiene
description: Enforce JIRA hygiene standards on one or many issues — sets story points, team, components, activity type, fix version, and labels. Also links CVE/Vulnerability JIRA issues to their source GitHub repos. Use this skill when the user says "jira hygiene", "fix jira hygiene", "clean up jiras", "hygiene check", "check my jiras", "fix my sprint hygiene", "enforce jira standards", "link cve repos", "cve repo linker", or wants to ensure their JIRA tickets have required fields filled in correctly. Also trigger when the user mentions missing story points, wrong team, missing components, activity type on JIRA issues, or linking CVEs to source repos. Trigger on "/jira-hygiene".
---

# JIRA Hygiene

Audit and fix JIRA issues so every ticket meets the team's field standards. Works on a single issue, all issues in a sprint, or all issues assigned to the user.

## Prerequisites

Three environment variables must be set (typically via `~/.secrets` sourced by the shell):

| Variable | Purpose |
|----------|----------|
| `JIRA_BASE_URL` | Atlassian instance URL (e.g. `https://redhat.atlassian.net`) |
| `JIRA_EMAIL` | User's email for basic auth |
| `JIRA_API_TOKEN` | Atlassian API token |

If any is missing, abort with: *"Missing environment variable `<name>`. Set it in `~/.secrets` and re-source your shell."*

## Input Modes

Parse the user's request to determine which issues to process:

| User says | Mode | What to fetch |
|-----------|------|---------------|
| A JIRA ID (e.g. `RHOAIENG-1234` or `j-1234`) | **single** | That one issue |
| A sprint name or "current sprint" | **sprint** | All issues assigned to user in that sprint |
| "all my jiras", "all assigned to me", etc. | **all** | All open issues assigned to user |

Normalize short-form IDs: `j-1234` → `RHOAIENG-1234`.

## Fetching Issues

Use curl with basic auth for all JIRA API calls:
```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H "Content-Type: application/json" \
  "$JIRA_BASE_URL/rest/api/3/..."
```

### Single issue
```
GET /rest/api/3/issue/{key}?fields=summary,description,status,components,labels,fixVersions,customfield_10028,customfield_10001,customfield_10464,customfield_10020,issuetype
```

### Sprint issues
```
GET /rest/api/3/search/jql?jql=assignee=currentUser() AND sprint="{sprintName}" AND project=RHOAIENG&fields=summary,description,status,components,labels,fixVersions,customfield_10028,customfield_10001,customfield_10464,customfield_10020,issuetype&maxResults=50
```
If user says "current sprint", use `sprint in openSprints()` instead of a named sprint.

### All assigned issues
```
GET /rest/api/3/search/jql?jql=assignee=currentUser() AND project=RHOAIENG AND statusCategory != Done&fields=summary,description,status,components,labels,fixVersions,customfield_10028,customfield_10001,customfield_10464,customfield_10020,issuetype&maxResults=100
```

Handle pagination if `total` exceeds `maxResults` — loop with `startAt` increments.

## Field Reference

| Field | API path | Custom field ID |
|-------|----------|------------------|
| Story Points | `customfield_10028` | Numeric (1, 2, 3, 5, 8) |
| Team | `customfield_10001` | Object with `name` |
| Components | `components` | Array of `{name}` |
| Activity Type | `customfield_10464` | Object with `value` |
| Sprint | `customfield_10020` | Array of sprint objects |
| Fix Versions | `fixVersions` | Array of `{name}` |
| Labels | `labels` | Array of strings |
| Status | `status.statusCategory.name` | Category: "To Do", "In Progress", "Done" |

## Core Hygiene Rules

For each issue, check these rules in order. This is the reusable logic — every input mode feeds issues through these same checks.

### Rule 1: Story Points
- **Required**: Always
- **Check**: `customfield_10028` is one of `1, 2, 3, 5, 8`
- **If missing or invalid**: Read the issue summary and description. Suggest a value based on apparent complexity:
  - 1 = trivial config change or typo fix
  - 2 = small, well-scoped task
  - 3 = moderate task with clear scope
  - 5 = significant work spanning multiple files or components
  - 8 = large effort, multi-day work
- Present the suggestion to the user for confirmation before setting.

### Rule 2: Team
- **Required**: Always
- **Check**: `customfield_10001.name` equals `"RHOAI Model Server and Serving Metrics"`
- **If wrong or missing**: Set to `{"id": "ec74d716-af36-4b3c-950f-f79213d08f71-2896"}`
- This is deterministic — no user input needed.

### Rule 3: Components
- **Required**: Always
- **Check**: `components` array contains an entry with `name` = `"Model Serving"`
- **If missing**: Add `{"name": "Model Serving"}` to the components array. Preserve any existing components.
- This is deterministic — no user input needed.

### Rule 4: Activity Type
- **Required**: Always
- **Check**: `customfield_10464.value` is one of the valid values
- **Valid values**:
  - `"Tech Debt & Quality"` — CVE fixes, bug fixes, refactoring, test improvements, CI/CD work, dependency bumps, code cleanup, flaky test fixes
  - `"New Features"` — new functionality, feature enhancements, new API endpoints, new integrations
  - `"Learning and Enablement"` — documentation, training, onboarding, knowledge sharing, runbook creation
- **If missing or wrong**: Infer the correct value from the issue summary and description. Consider keywords:
  - CVE, vulnerability, bug, fix, refactor, cleanup, debt, test, CI, flaky, upgrade, bump, deprecat → **Tech Debt & Quality**
  - feature, add, new, implement, enable, support, integrate, enhance → **New Features**
  - doc, document, learn, train, onboard, runbook, guide, tutorial → **Learning and Enablement**
- Present the inference to the user for confirmation before setting.

### Rule 5: Fix Version (conditional)
- **Required**: Only if the issue is in a sprint (`customfield_10020` is non-empty)
- **Check**: `fixVersions` array is non-empty
- **If missing**: Fetch available versions from the project:
  ```
  GET /rest/api/3/project/RHOAIENG/versions?status=unreleased&orderBy=-releaseDate
  ```
  Check if other issues in the same sprint have a fix version — if so, suggest that version. Otherwise present the most recent unreleased versions and ask the user to pick one.

### Rule 6: Labels (conditional)
- **Required**: Only if status category is `"In Progress"`
- **Check**: `labels` array contains both `"ai-first"` and `"ai-accelerated-fix"`
- **If missing**: Add the missing labels. Preserve any existing labels.
- This is deterministic — no user input needed.

## Workflow

### Step 1: Validate Prerequisites
Check that `JIRA_BASE_URL`, `JIRA_EMAIL`, and `JIRA_API_TOKEN` are set in the environment.

### Step 2: Parse Input and Fetch Issues
Determine the input mode, construct the appropriate API call, and fetch all target issues.

### Step 3: Audit All Issues
Run every issue through the Core Hygiene Rules. For each issue, build a findings list:

```
RHOAIENG-1234: "Fix storage initializer crash"
  ✅ Story Points: 3
  ❌ Team: missing → will set to "RHOAI Model Server and Serving Metrics"
  ✅ Components: Model Serving
  ❌ Activity Type: missing → suggest "Tech Debt & Quality" (CVE fix)
  ✅ Fix Version: rhoai-3.5.EA2
  ⚠️  Labels: in progress but missing "ai-first", "ai-accelerated-fix"
```

Use ✅ for passing, ❌ for failing (will fix), ⚠️ for conditional rules that apply.

### Step 4: Present Findings
Show the full audit table to the user. For fields that need user input (story points, activity type, fix version), collect answers.

When processing multiple issues, batch the questions efficiently:
- Group issues that need the same type of input
- For activity type, show the inferred value and ask for confirmation as a batch: *"I'll set these activity types — correct? (Y to confirm, or list corrections)"*
- For story points, show suggestions for all issues and ask for confirmation similarly

### Step 5: Apply Fixes
For each issue with findings, build a single PUT request that updates all failing fields at once:

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -X PUT \
  -H "Content-Type: application/json" \
  "$JIRA_BASE_URL/rest/api/3/issue/{key}" \
  -d '{"fields": { ... }}'
```

**Field update formats:**
```json
{
  "customfield_10028": 3,
  "customfield_10001": {"id": "ec74d716-af36-4b3c-950f-f79213d08f71-2896"},
  "components": [{"name": "Model Serving"}],
  "customfield_10464": {"value": "Tech Debt & Quality"},
  "fixVersions": [{"name": "rhoai-3.5.EA2"}],
  "labels": ["ai-first", "ai-accelerated-fix", "existing-label"]
}
```

Only include fields that need changing. For `components` and `labels`, merge with existing values — never overwrite.

For labels specifically, use the label-add endpoint to avoid overwriting:
```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -X PUT \
  -H "Content-Type: application/json" \
  "$JIRA_BASE_URL/rest/api/3/issue/{key}" \
  -d '{"update": {"labels": [{"add": "ai-first"}, {"add": "ai-accelerated-fix"}]}}'
```

For components, similarly use the update syntax to add without replacing:
```bash
-d '{"update": {"components": [{"add": {"name": "Model Serving"}}]}}'
```

If the issue needs both `fields` updates and `update` operations, combine them in one request:
```json
{
  "fields": {
    "customfield_10028": 3,
    "customfield_10001": {"id": "ec74d716-af36-4b3c-950f-f79213d08f71-2896"},
    "customfield_10464": {"value": "Tech Debt & Quality"},
    "fixVersions": [{"name": "rhoai-3.5.EA2"}]
  },
  "update": {
    "labels": [{"add": "ai-first"}, {"add": "ai-accelerated-fix"}],
    "components": [{"add": {"name": "Model Serving"}}]
  }
}
```

Check the HTTP response code — 204 means success. If a request fails, report the error and continue with remaining issues.

### Step 6: Summary
After all updates, show a summary:

```
JIRA Hygiene Complete — 5 issues processed
  ✅ 3 issues updated successfully
  ⏭️  2 issues already clean
  ❌ 0 failures
```

For each updated issue, list what changed.
