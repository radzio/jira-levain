# jira-levain

Claude Code plugin that gathers Jira ticket context — hierarchy, linked issues, comments, Confluence docs — and prepares a structured implementation or debugging plan. Levain is the sourdough starter that kicks off the entire bake.

## What it does

1. Fetches full Jira ticket details via MCP (summary, description, status, priority, comments)
2. Traverses the hierarchy upward (subtask → story → epic → initiative) for full context
3. Gathers linked issues and their relationship types
4. Reads linked Confluence pages for specs, designs, and technical docs
5. Produces a tailored plan — debugging plan for bugs, implementation plan for features

## Install

### Via marketplace (recommended)

```bash
claude plugin marketplace add https://github.com/radzio/plugin-patisserie
claude plugin install jira-levain
```

### Standalone

```bash
claude plugin add https://github.com/radzio/jira-levain
```

## Usage

```
/levain PROJ-123
```

Pass any Jira ticket key. The plugin will gather context and walk you through clarifying questions before generating a plan.

## Prerequisites

You need a Jira (and optionally Confluence) MCP server. Pick one:

### Option 1: Atlassian plugin for Claude Code (recommended)

```bash
claude plugin add atlassian
```

Official plugin — handles Jira + Confluence, authenticates via OAuth. No API tokens needed.

### Option 2: mcp-atlassian via uvx

```bash
claude mcp add atlassian -s user -- uvx mcp-atlassian
```

Community package from [sooperset/mcp-atlassian](https://github.com/sooperset/mcp-atlassian). Supports Cloud and Server/Data Center. Requires API tokens — set in `.mcp.json`:

```json
{
  "mcpServers": {
    "atlassian": {
      "command": "uvx",
      "args": ["mcp-atlassian"],
      "env": {
        "JIRA_URL": "https://your-org.atlassian.net",
        "JIRA_USERNAME": "you@example.com",
        "JIRA_API_TOKEN": "your-api-token",
        "CONFLUENCE_URL": "https://your-org.atlassian.net/wiki",
        "CONFLUENCE_USERNAME": "you@example.com",
        "CONFLUENCE_API_TOKEN": "your-api-token"
      }
    }
  }
}
```

### Option 3: Atlassian remote MCP server

```bash
claude mcp add --transport sse atlassian https://mcp.atlassian.com/v1/sse
```

Official remote server from [atlassian/atlassian-mcp-server](https://github.com/atlassian/atlassian-mcp-server). Cloud only, authenticates via OAuth.
