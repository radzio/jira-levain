---
description: Starter for your next bake — gather Jira context and prepare an implementation plan
allowed-tools: mcp__jira__*, mcp__confluence__*, mcp__mcp-atlassian__*, mcp__atlassian__*, Read, Grep, Glob
---

# Levain — Jira Context Gatherer & Plan Builder

You are preparing a structured implementation or debugging plan from a Jira ticket. The user has provided a ticket key as an argument: `$ARGUMENTS`. Follow these steps exactly.

## Step 1: Fetch the Ticket

Call the Jira MCP server to get the full issue details for `$ARGUMENTS`.

Try these MCP tool name variants in order until one works:
1. `mcp__jira__get_issue`
2. `mcp__atlassian__get_issue`
3. `mcp__mcp-atlassian__get_issue`
4. `mcp__jira__jira_get_issue`
5. `mcp__atlassian__jira_get_issue`
6. `mcp__mcp-atlassian__jira_get_issue`

If none work, try listing available MCP tools to find the correct Jira tool name. If no Jira MCP tools are available at all, stop and display:

```
## Jira MCP server not configured

To use /levain, you need a Jira MCP server. Add one to your Claude Code MCP settings:

Common packages:
- mcp-atlassian: `uvx mcp-atlassian`
- @anthropic/mcp-atlassian

See the plugin README for setup details.
```

Once you have the issue, extract:
- **Key**, **Summary**, **Type** (bug, story, task, epic, subtask, etc.)
- **Status**, **Priority**, **Assignee**, **Reporter**
- **Description** (full text)
- **Parent key** (if any)
- **Linked issues** (keys and relationship types)
- **Comments** (all)
- **Labels**, **Components**, **Sprint** (if available)

## Step 2: Traverse Hierarchy Up

If the ticket has a parent, fetch the parent issue. Continue walking up parent links for up to 3 levels total (e.g., subtask → story → epic → initiative).

For each parent, extract:
- Key, summary, type, status, description
- Comments (all)

Build a hierarchy chain, for example:
```
PROJ-10 (Initiative) → PROJ-50 (Epic) → PROJ-123 (Story) → PROJ-456 (Subtask) ← YOU ARE HERE
```

If there is no parent, note that the ticket is a top-level issue.

## Step 3: Gather Linked Issues

For each linked issue on the target ticket, fetch the issue details (1 level only — do not recurse into their links).

Record:
- Key, summary, type, status
- Relationship type: **blocks**, **is blocked by**, **relates to**, **duplicates**, **is duplicated by**, **causes**, **is caused by**, etc.
- Brief description excerpt (first ~200 chars)

If there are no linked issues, note that and move on.

## Step 4: Check Confluence Links

### 4a: Remote links

Try to fetch remote links for the target ticket and each parent using variants:
1. `mcp__jira__get_issue_remote_links`
2. `mcp__atlassian__get_issue_remote_links`
3. `mcp__mcp-atlassian__get_issue_remote_links`
4. `mcp__jira__jira_get_issue_remote_links`
5. `mcp__atlassian__jira_get_issue_remote_links`

If remote links are available, identify any that point to Confluence (URLs containing `/wiki/` or `confluence`).

### 4b: Inline Confluence URLs

Scan the description and comments of the target ticket and its parents for Confluence URLs (patterns like `https://...atlassian.net/wiki/...` or `https://confluence...`).

### 4c: Fetch Confluence pages

For each Confluence URL found (from remote links or inline), try to fetch the page content using:
1. `mcp__confluence__get_page`
2. `mcp__atlassian__confluence_get_page`
3. `mcp__mcp-atlassian__confluence_get_page`
4. `mcp__confluence__confluence_get_page`
5. `mcp__atlassian__get_page`

Extract the page ID from the URL if needed. If no Confluence MCP tools are available, note this gracefully:

```
ℹ Confluence links found but no Confluence MCP server configured — skipping page content.
```

List the URLs so the user can reference them manually.

## Step 5: Compile Comments

From the target ticket and its direct parent (if any), compile all comments and extract key insights:

- **Technical decisions** — architecture choices, library selections, approach agreements
- **Scope clarifications** — what's in/out of scope, changed requirements
- **Blockers** — identified blockers, dependencies, waiting-on items
- **Acceptance criteria** — explicit or implicit acceptance criteria mentioned in comments
- **Code references** — file paths, class names, function names, PRs, commits mentioned
- **Previous attempts** — any past approaches that were tried and abandoned, with reasons

Summarize the key insights. Don't just dump raw comments — distill what matters for implementation.

## Step 6: Display Briefing

Present a structured summary:

```
## 🧑‍🍳 Levain Briefing: {KEY} — {Summary}

**Type:** {type} · **Status:** {status} · **Priority:** {priority}
**Assignee:** {assignee} · **Reporter:** {reporter}
**Labels:** {labels} · **Components:** {components} · **Sprint:** {sprint}

### Hierarchy
{hierarchy chain, or "Top-level issue" if none}

### Description
{full description text}

### Linked Issues
{table or list of linked issues with relationship types and statuses}

### Confluence Context
{summaries of fetched Confluence pages, or "No Confluence links found"}

### Key Insights from Comments
{distilled insights from Step 5, organized by category}

### Acceptance Criteria
{extracted acceptance criteria from description + comments, or "None explicitly stated"}
```

## Step 7: Ask Clarifying Questions

**If the ticket type is Epic:**
Do NOT generate a plan. Instead, display:

```
### Epic Overview

This is an epic — here's the scope:

{list child tickets with their keys, summaries, types, statuses, and assignees}

Epics are too broad for a single implementation plan. Pick a child ticket to plan:

> Run `/levain {CHILD-KEY}` to plan a specific ticket.
```

Then stop.

**If the ticket type is Bug**, ask clarifying questions using AskUserQuestion. Skip any questions already answered by the gathered context (description, comments, linked issues). Select from:

- Can you reproduce this bug? What are the reproduction steps?
- Which environment does this occur in? (production / staging / local dev)
- Is this a regression? When did it last work correctly?
- Does this affect all users or a specific subset?
- Any recent deploys or config changes that might be related?
- Is there a known workaround?

**If the ticket type is Story, Task, Feature, or Subtask**, ask clarifying questions using AskUserQuestion. Skip any questions already answered by the gathered context. Select from:

- Do you have a preferred technical approach or pattern in mind?
- Which codebase areas or files are most relevant?
- Are there constraints I should know about? (backward compatibility, performance requirements, library preferences)
- Any edge cases you're particularly concerned about?
- External dependencies to coordinate with other teams?
- What level of test coverage do you expect? (unit, integration, e2e)

Present only the questions that aren't already answered by the context gathered in Steps 1–5. If all questions are answered by the context, skip this step and note that the context is comprehensive.

## Step 8: Generate Plan

Based on everything gathered, produce a structured plan tailored to the ticket type.

### For Bugs → Debugging Plan

```
## 🔍 Debugging Plan: {KEY} — {Summary}

### Bug Summary
{1-2 sentence summary of the bug and its impact}

### Reproduction Steps
{steps to reproduce, from user input or ticket description}

### Affected Area Analysis
{which parts of the codebase are likely involved, based on description + code references from comments}

### Hypotheses (ranked by likelihood)
1. **{Hypothesis}** — {reasoning based on gathered context}
2. **{Hypothesis}** — {reasoning}
3. **{Hypothesis}** — {reasoning}

### Investigation Steps
{ordered steps to systematically verify/eliminate each hypothesis}

1. {step — what to check, what to look for, which files/logs to examine}
2. {step}
3. {step}

### Suggested Fix Approach
{based on the most likely hypothesis, outline the fix strategy}

### Testing Plan
- {how to verify the fix}
- {regression tests to add}
- {manual verification steps}

### Risks & Considerations
- {potential side effects}
- {related areas that might break}
- {rollback strategy if needed}
```

### For Features/Stories/Tasks → Implementation Plan

```
## 🏗️ Implementation Plan: {KEY} — {Summary}

### Requirements Summary
{distilled requirements from description + hierarchy context + Confluence docs}

### Acceptance Criteria
- [ ] {criterion}
- [ ] {criterion}
- [ ] {criterion}

### Technical Approach
{high-level approach, informed by any decisions from comments or Confluence docs}

### Implementation Steps
{ordered steps, each with specific files to modify/create}

1. **{Step title}**
   - Files: `{path/to/file}`
   - What: {specific changes}

2. **{Step title}**
   - Files: `{path/to/file}`
   - What: {specific changes}

### Edge Cases & Error Handling
- {edge case and how to handle it}
- {error scenario and recovery strategy}

### Dependencies
- {external services, other tickets, or team coordination needed}

### Testing Approach
- **Unit tests:** {what to test}
- **Integration tests:** {what to test}
- **Manual verification:** {steps}

### Risks & Open Questions
- {risk and mitigation}
- {unresolved question that might affect implementation}
```

---

**IMPORTANT:** This command is **read-only**. Gather context and produce a plan, but do NOT modify any project files. Present the plan and let the user decide how to proceed with implementation.
