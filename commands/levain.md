---
description: Starter for your next bake — gather Jira context and prepare an implementation plan
allowed-tools: mcp__jira__*, mcp__confluence__*, mcp__mcp-atlassian__*, mcp__atlassian__*, mcp__plugin_atlassian_atlassian__*, Read, Grep, Glob
---

# Levain — Jira Context Gatherer & Plan Builder

You are preparing a structured implementation or debugging plan from a Jira ticket. The user has provided a ticket key as an argument: `$ARGUMENTS`. Follow these steps exactly.

---

## Step 1: Discover MCP Tools & Fetch Target Ticket

### 1a: Tool discovery

You need Jira MCP tools. Different users register their Atlassian MCP server under different names. Common tool patterns:

| Server name | Tool pattern | Example |
|-------------|-------------|---------|
| `plugin_atlassian_atlassian` | `mcp__plugin_atlassian_atlassian__getJiraIssue` | Atlassian plugin for Claude Code |
| `atlassian` | `mcp__atlassian__getJiraIssue` | mcp-atlassian via uvx |
| `jira` | `mcp__jira__getJiraIssue` or `mcp__jira__get_issue` | Standalone Jira MCP |
| `mcp-atlassian` | `mcp__mcp-atlassian__getJiraIssue` | mcp-atlassian (hyphenated) |

Try to fetch the issue for `$ARGUMENTS`. If the first tool name fails, try the next pattern. Once you find the working pattern, use it for all subsequent calls.

If no Jira MCP tools are available at all, stop and display:

```
## Jira MCP server not configured

To use /levain, you need a Jira MCP server. Add one to your Claude Code MCP settings.

Common packages:
- Atlassian plugin: `claude plugin add atlassian` (recommended)
- mcp-atlassian: `uvx mcp-atlassian`

See the plugin README for setup details.
```

### 1b: Fetch target ticket

Once you find the working tool, fetch full issue details for `$ARGUMENTS`. You also need the **cloudId** — call the accessible resources tool (e.g., `getAccessibleAtlassianResources`) to get it if required by the tools.

Extract and store:
- **Key**, **Summary**, **Type** (bug, story, task, epic, subtask, etc.)
- **Status**, **Priority**, **Assignee**, **Reporter**
- **Description** (full text)
- **Parent key** (if any — from `parent` field or `parent.key`)
- **Linked issue keys** (from `issuelinks` field) with relationship types
- **Comments** (all)
- **Labels**, **Components**, **Fix Version**, **Sprint** (if available)

---

## Step 2: Build the Ticket Graph (Deep Traversal)

Explore the ticket's full context by traversing the graph in three phases. Track all visited ticket keys to avoid fetching the same ticket twice.

### Phase A: Walk parents upward

If the target ticket has a parent, fetch it. Continue walking parent links up to **3 levels** (e.g., subtask → story → epic → initiative). For each parent, extract the same fields as Step 1b.

Build a hierarchy chain:
```
PROJ-10 (Initiative) → PROJ-50 (Epic) → PROJ-123 (Story) → PROJ-456 (Subtask) ← YOU ARE HERE
```

### Phase B: Gather all linked tickets

Collect linked issue keys from the **target ticket AND every parent**. Deduplicate. Then fetch all of them **in parallel**.

For each linked ticket, extract:
- Key, summary, type, status, priority, description (first ~300 chars)
- Relationship type and direction: **blocks / is blocked by**, **relates to**, **duplicates / is duplicated by**, **causes / is caused by**, **clones / is cloned by**, etc.
- Comments (all)
- Their own linked issue keys (record but do NOT fetch — this is the boundary)

### Phase C: Enrich every ticket with remote links & Confluence

For **every ticket in the graph** (target + parents + linked), do the following **in parallel**:

1. **Fetch remote links** — call the remote links tool (e.g., `getJiraIssueRemoteIssueLinks`). Identify any Confluence URLs (containing `/wiki/` or `confluence`).

2. **Scan text for Confluence URLs** — search the ticket's description and comments for URLs matching `https://...atlassian.net/wiki/...` or `https://confluence...`.

3. **Collect all unique Confluence page IDs/URLs** across the entire graph.

### Phase D: Fetch Confluence pages

Deduplicate all discovered Confluence URLs. For each unique page, fetch content using the Confluence tool (e.g., `getConfluencePage`). **Fetch in parallel.**

To extract the page ID from a Confluence URL:
- Pattern: `.../pages/{pageId}/...` → use `{pageId}`
- Pattern: `.../spaces/{spaceKey}/pages/{pageId}` → use `{pageId}`

If Confluence tools are not available, note gracefully:
```
ℹ Confluence links found but no Confluence MCP configured — listing URLs for manual reference.
```

And list the URLs.

---

## Step 3: Compile Insights

Process ALL gathered data across the entire ticket graph. Extract and organize:

### From comments (target + parents + linked tickets):
- **Technical decisions** — architecture choices, library selections, approach agreements
- **Scope clarifications** — what's in/out of scope, changed requirements
- **Blockers** — identified blockers, dependencies, waiting-on items
- **Acceptance criteria** — explicit or implicit criteria from any ticket in the graph
- **Code references** — file paths, class names, function names, PRs, commits, branch names
- **Previous attempts** — approaches tried and abandoned, with reasons
- **Environment details** — relevant environment info, config, versions

### From Confluence pages:
- **Specs & designs** — summarize relevant sections
- **Technical documentation** — architecture docs, API specs, data models
- **Decision records** — ADRs or similar documents explaining past decisions

### From the link graph:
- **Blockers & dependencies** — tickets that block this work or must be coordinated with
- **Related work** — tickets working on the same area that might conflict
- **Duplicates** — similar or duplicate tickets that provide additional context
- **Second-degree links** — notable linked tickets discovered on parents/linked tickets (keys only, not fetched)

Don't dump raw data — distill what matters for implementation. Attribute insights to their source ticket.

---

## Step 4: Display Briefing

Present a structured summary:

```
## Levain Briefing: {KEY} — {Summary}

**Type:** {type} · **Status:** {status} · **Priority:** {priority}
**Assignee:** {assignee} · **Reporter:** {reporter}
**Labels:** {labels} · **Components:** {components} · **Sprint:** {sprint}

### Hierarchy
{hierarchy chain, or "Top-level issue" if none}

### Description
{full description text}

### Linked Issues
| Key | Type | Relationship | Status | Summary |
|-----|------|-------------|--------|---------|
| ... | ...  | ...         | ...    | ...     |

{If linked tickets have their own notable links, mention them:}
> PROJ-789 (linked) also links to PROJ-999, PROJ-1000 — potentially related.

### Confluence Context
{For each Confluence page: title, source ticket, and key content summary}
{Or "No Confluence links found across the ticket graph"}

### Key Insights from Comments
{Organized by category, with source attribution}

**Technical Decisions:**
- {insight} — from {TICKET-KEY} comment by @author

**Scope & Requirements:**
- {insight} — from {TICKET-KEY}

**Blockers & Dependencies:**
- {insight}

**Code References:**
- {file/class/function references found across comments}

**Previous Attempts:**
- {what was tried and why it didn't work}

### Acceptance Criteria
{Consolidated from description + comments across all tickets, or "None explicitly stated"}
```

---

## Step 5: Ask Clarifying Questions

**If the ticket type is Epic:**
Do NOT generate a plan. Instead, use JQL to search for child tickets (e.g., `"Epic Link" = {KEY} OR parent = {KEY}`). Display:

```
### Epic Overview

This is an epic — here's the scope:

| Key | Type | Status | Assignee | Summary |
|-----|------|--------|----------|---------|
| ... | ...  | ...    | ...      | ...     |

Epics are too broad for a single implementation plan. Pick a child ticket to plan:

> Run `/levain {CHILD-KEY}` to plan a specific ticket.
```

Then stop.

**If the ticket type is Bug**, ask clarifying questions using AskUserQuestion. **Skip any questions already answered** by the gathered context (description, comments, linked issues, Confluence). Select from:

- Can you reproduce this bug? What are the reproduction steps?
- Which environment does this occur in? (production / staging / local dev)
- Is this a regression? When did it last work correctly?
- Does this affect all users or a specific subset?
- Any recent deploys or config changes that might be related?
- Is there a known workaround?

**If the ticket type is Story, Task, Feature, or Subtask**, ask clarifying questions using AskUserQuestion. **Skip any questions already answered.** Select from:

- Do you have a preferred technical approach or pattern in mind?
- Which codebase areas or files are most relevant?
- Are there constraints I should know about? (backward compatibility, performance requirements, library preferences)
- Any edge cases you're particularly concerned about?
- External dependencies to coordinate with other teams?
- What level of test coverage do you expect? (unit, integration, e2e)

If all questions are answered by context, skip this step and note that the context is comprehensive.

---

## Step 6: Generate Plan

Based on everything gathered, produce a structured plan tailored to the ticket type.

### For Bugs → Debugging Plan

```
## Debugging Plan: {KEY} — {Summary}

### Bug Summary
{1-2 sentence summary of the bug and its impact}

### Reproduction Steps
{steps to reproduce, from user input or ticket description}

### Affected Area Analysis
{which parts of the codebase are likely involved, informed by code references from across the ticket graph}

### Hypotheses (ranked by likelihood)
1. **{Hypothesis}** — {reasoning, citing evidence from tickets/comments/confluence}
2. **{Hypothesis}** — {reasoning}
3. **{Hypothesis}** — {reasoning}

### Investigation Steps
{ordered steps to systematically verify/eliminate each hypothesis}

1. {step — what to check, what to look for, which files/logs to examine}
2. {step}
3. {step}

### Suggested Fix Approach
{based on the most likely hypothesis, outline the fix strategy}
{reference any related tickets or previous attempts that inform this approach}

### Testing Plan
- {how to verify the fix}
- {regression tests to add}
- {manual verification steps}

### Risks & Considerations
- {potential side effects, informed by linked ticket context}
- {related areas that might break}
- {rollback strategy if needed}
```

### For Features/Stories/Tasks → Implementation Plan

```
## Implementation Plan: {KEY} — {Summary}

### Requirements Summary
{distilled requirements from description + hierarchy context + Confluence docs + linked tickets}

### Acceptance Criteria
- [ ] {criterion}
- [ ] {criterion}
- [ ] {criterion}

### Technical Approach
{high-level approach, informed by decisions from comments, Confluence docs, and related tickets}
{reference any technical decisions or constraints discovered in the graph}

### Implementation Steps
{ordered steps, each with specific files to modify/create}

1. **{Step title}**
   - Files: `{path/to/file}`
   - What: {specific changes}

2. **{Step title}**
   - Files: `{path/to/file}`
   - What: {specific changes}

### Edge Cases & Error Handling
- {edge case, especially ones mentioned in comments across the graph}
- {error scenario and recovery strategy}

### Dependencies
- {blocking tickets and their status}
- {external services or team coordination}
- {related tickets working on the same area}

### Testing Approach
- **Unit tests:** {what to test}
- **Integration tests:** {what to test}
- **Manual verification:** {steps}

### Risks & Open Questions
- {risk and mitigation, informed by previous attempts found in comments}
- {unresolved question that might affect implementation}
```

---

**IMPORTANT:** This command is **read-only**. Gather context and produce a plan, but do NOT modify any project files. Present the plan and let the user decide how to proceed with implementation.
