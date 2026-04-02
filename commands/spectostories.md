---
description: "Create Jira Epic and Stories from spec.md user stories — no tasks.md required"
tools:
  # Server name is configurable via mcp_server in jira-config.yml (default: "atlassian")
  - '{mcp_server}/createJiraIssue'
  - '{mcp_server}/editJiraIssue'
  - '{mcp_server}/searchJiraIssuesUsingJql'
  - '{mcp_server}/getJiraIssue'
  - '{mcp_server}/getJiraProjectIssueTypesMetadata'
---

# Create Jira Stories from Spec

This command creates a Jira Epic and one Story per user story from `spec.md` — **no `tasks.md` required**.

It is designed to run immediately after `/speckit.specify`, so stories are in Jira and available for review before any technical planning begins.

**How it differs from `/speckit.jira.specstoissues`:**

| | `spectostories` | `specstoissues` |
|---|---|---|
| **Source** | `spec.md` user stories | `tasks.md` phase headers |
| **Requires** | `spec.md` only | `spec.md` + `tasks.md` |
| **When to run** | After `/speckit.specify` | After `/speckit.tasks` |
| **Stories represent** | User-facing acceptance stories | Implementation phases |

## Prerequisites

1. MCP server providing Jira tools configured and running (server name configured in jira-config.yml)
2. Jira configuration file exists: `.specify/extensions/jira/jira-config.yml`
3. Specification directory with `spec.md` in `specs/<spec-name>/`

## User Input

$ARGUMENTS

Accepts optional `--spec <name>` argument to specify which specification to use.
If not provided, auto-detects from current directory or available specs.

## Steps

### 1. Detect Specification Directory

Determine which specification to use (in order of priority):

1. `--spec <name>` argument
2. Git branch name (if matches a spec directory)
3. Current directory (if inside `specs/<name>/`)
4. Single spec (if only one exists)

Read the specification directory and validate that `spec.md` exists.

```
❌ No spec found. Run /speckit.specify first to create a specification.
```

### 2. Load Jira Configuration

Load the Jira configuration from `.specify/extensions/jira/jira-config.yml`:

**Spec Stories Configuration** (with fallbacks to artifact mapping):

| Setting | Config path | Fallback | Default |
|---|---|---|---|
| Epic issue type | `spectostories.spec_artifact` | `mapping.spec_artifact` | `"Epic"` |
| Story issue type | `spectostories.story_artifact` | `mapping.phase_artifact` | `"Story"` |
| Epic–Story relationship | `spectostories.spec_story_relationship` | `mapping.relationships.spec_phase` | `"Parent"` |
| Epic labels | `defaults.spec.labels` | — | `[]` |
| Epic custom fields | `defaults.spec.custom_fields` | — | `{}` |
| Story labels | `defaults.story.labels` | `defaults.phase.labels` | `[]` |
| Story custom fields | `defaults.story.custom_fields` | `defaults.phase.custom_fields` | `{}` |

**Environment variable overrides** (checked first):
- `SPECKIT_JIRA_MCP_SERVER`
- `SPECKIT_JIRA_PROJECT_KEY`
- `SPECKIT_JIRA_SPEC_ARTIFACT` → Epic issue type
- `SPECKIT_JIRA_STORY_ARTIFACT` → Story issue type
- `SPECKIT_JIRA_SPEC_STORY_RELATIONSHIP` → Epic–Story link type

### 3. Check for Existing Issues

Scan `spec.md` for an existing `## Jira` section.

If found, display the existing Epic and Story links, then ask:

- **A) Skip** — stop here (issues already exist)
- **B) Update** — skip the Epic and any Stories already listed; only create new ones added since the last run
- **C) Replace** — create everything again (warn: will create duplicates)

Also check whether `specs/<name>/jira-mapping.json` exists and mention it if present.

### 4. Get Project Issue Type Metadata

Call `getJiraProjectIssueTypesMetadata` for the project to:

- Confirm exact `issueTypeName` strings for the configured Epic and Story types
- Identify any required type-specific fields (e.g. custom "Epic Name" field)

If the project key returns an error, fail with a clear message directing the user to fix `jira-config.yml`.

### 5. Parse spec.md

Read the full `spec.md` and extract:

**Feature metadata:**
- **Title**: text of the first `# ` heading
- **Branch name**: from `**Feature Branch**:` line, or from `git branch --show-current`
- **Overview**: introductory content before the first `##` section (used for the Epic description)

**User stories** — all `### User Story N — [Title] (Priority: Px)` headings and their body:

| Field | Source in spec.md |
|---|---|
| Story number | `N` in `User Story N` |
| Title | text between ` — ` and ` (Priority:` |
| Priority | `P1`, `P2`, `P3` etc. |
| Journey paragraph | first prose paragraph after the heading |
| Why this priority | content after `**Why this priority**:` |
| Independent Test | content after `**Independent Test**:` (if present) |
| Acceptance Scenarios | all `**Given**…**When**…**Then**…` lines |

### 6. Create the Epic

```
Tool: {mcp_server}/createJiraIssue
Parameters:
  projectKey: {project.key}
  issueTypeName: {spec_artifact}
  summary: {feature_title}
  description: |
    {overview_paragraph}

    ---
    Spec: specs/{spec_name}/spec.md (branch: {branch_name})
  additional_fields: {defaults.spec.custom_fields}
```

Include any required type-specific fields discovered in step 4.

Store the created Epic key (e.g. `PROJ-100`) for linking stories.

Display:
```
✅ Created Epic: PROJ-100 — My Feature
   https://your-jira.atlassian.net/browse/PROJ-100
```

### 7. Create One Story Per User Story

For each user story extracted in step 5:

**Step 7a: Create the Story**

```
Tool: {mcp_server}/createJiraIssue
Parameters:
  projectKey: {project.key}
  issueTypeName: {story_artifact}
  summary: {story_title}
  description: |
    Priority: {priority}

    {journey_paragraph}

    Why this priority: {why_this_priority}

    Independent Test: {independent_test}

    Acceptance Scenarios:
    1. Given ... When ... Then ...
    2. Given ... When ... Then ...

    ---
    Spec: specs/{spec_name}/spec.md (branch: {branch_name})
  additional_fields: {defaults.story.custom_fields}
```

**Step 7b: Link Story to Epic**

Immediately after creating each Story, link it to the Epic based on the configured relationship:

| `spec_story_relationship` value | Action |
|---|---|
| `"Parent"` | `editJiraIssue` with `fields: {"parent": {"key": "{epic_key}"}}` — sets proper breadcrumb hierarchy |
| `"Epic Link"` | `editJiraIssue` with the Epic Link custom field set to the Epic key |
| `"Relates"` / `"Blocks"` / etc. | Create issue link from Story to Epic |
| `"none"` | No link created |

> **Important**: For `"Parent"`, use `editJiraIssue` with `fields: {"parent": {"key": "..."}}` — do **not** use `createIssueLink`, which only creates a "Linked work items" entry rather than a true Jira parent.

Display:
```
  ✅ PROJ-101 — Scan Top Stories on App Open (P1)
  ✅ PROJ-102 — Live and Breaking News Awareness (P1)
  ✅ PROJ-103 — Quick Access via Story Bubbles (P2)
  ...
```

### 8. Save Mapping File

Write `specs/<name>/jira-mapping.json`:

```json
{
  "created_at": "<ISO-8601 timestamp>",
  "spec": "<spec-name>",
  "project": "<project_key>",
  "jira_base_url": "https://<your-jira>.atlassian.net",
  "mode": "spec-stories",
  "epic": {
    "key": "<epic_key>",
    "summary": "<feature_title>",
    "url": "<jira_base_url>/browse/<epic_key>"
  },
  "stories": [
    {
      "story_number": 1,
      "key": "<story_key>",
      "summary": "<story_title>",
      "priority": "P1",
      "url": "<jira_base_url>/browse/<story_key>"
    }
  ]
}
```

The `"mode": "spec-stories"` distinguishes this mapping from `"2-level"` and `"3-level"` mappings created by `specstoissues`.

### 9. Update spec.md

Append (or replace the existing `## Jira` section) at the end of `spec.md`:

```markdown
## Jira

**Project**: {project_key}
**Epic**: [{feature_title}]({epic_url})
**Created**: {YYYY-MM-DD}

| Story | Ticket | Priority |
|---|---|---|
| {story_title} | [{story_key}]({story_url}) | {priority} |
```

### 10. Report

```
✅ Epic:    PROJ-100 — {feature_title}
            {epic_url}

✅ Stories:
   PROJ-101 — {story_1_title} (P1)
   PROJ-102 — {story_2_title} (P1)
   PROJ-103 — {story_3_title} (P2)
   ...

📁 Mapping: specs/{spec_name}/jira-mapping.json

Recommended next steps:
  1. Share the Epic link with stakeholders for review
  2. After review, run /speckit.clarify to resolve feedback
  3. Run /speckit.plan to begin technical planning
  4. After /speckit.tasks, run /speckit.jira.specstoissues for implementation sub-tasks
```
