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

- **Jira MCP server** (required) — provides ticket data via MCP tools. Common packages:
  - [`@anthropic/mcp-atlassian`](https://github.com/anthropic/mcp-atlassian)
  - [`mcp-atlassian`](https://github.com/sooperset/mcp-atlassian)

- **Confluence MCP server** (optional) — enables reading linked Confluence pages. Often bundled with the Jira MCP server above.

### MCP setup

Add a Jira MCP server to your Claude Code configuration (`.mcp.json` or project settings). Example:

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
