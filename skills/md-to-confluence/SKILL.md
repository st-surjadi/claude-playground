---
name: md-to-confluence
description: Publish a markdown file to a Confluence page using the Atlassian MCP server — handles MCP setup, page creation, and updates
---

# Markdown to Confluence

Publish markdown files to Confluence pages using the official Atlassian MCP server. Supports two modes: **setup** (configure MCP + create initial page) and **publish** (update an existing page).

## Prerequisites: Atlassian MCP Server

Before publishing, ensure the Atlassian MCP server is connected. Check if Confluence MCP tools are available (e.g., tools for creating/updating pages).

**If the MCP server is NOT set up**, run:

```bash
claude mcp add --transport sse atlassian https://mcp.atlassian.com/v1/sse
```

This triggers an OAuth 2.1 browser flow — the user authenticates with their Atlassian account and grants access. No API tokens or manual config needed.

**Alternative**: If the user prefers project-scoped config, add to `.mcp.json` in the project root:

```json
{
  "mcpServers": {
    "Atlassian": {
      "command": "npx",
      "args": ["-y", "mcp-remote@latest", "https://mcp.atlassian.com/v1/sse"]
    }
  }
}
```

After setup, restart Claude Code for the MCP server to be available.

## Mode 1: Setup

Use this mode when publishing a markdown file to Confluence for the first time.

### Steps

1. **Check MCP availability** — verify Confluence MCP tools are accessible. If not, guide the user through the prerequisite setup above and ask them to restart Claude Code.

2. **Ask the user** using AskUserQuestion:
   - "What is the Confluence Space key?" (e.g., `ENG`, `PLATFORM`)
   - "What is the parent page title?" (optional, e.g., `RFCs`)
   - "Or, if you have the parent page URL, just paste it and I'll extract the space and parent page automatically."

   If the user provides a Confluence page URL (e.g., `https://site.atlassian.net/wiki/spaces/SPACE/pages/123456/Page+Title`), extract the space key and page ID from the URL and use `getConfluencePage` to resolve the space ID and parent page — skip the space key and parent page title questions.

3. **Read the markdown file** and extract:
   - The title from the first `# heading`
   - The full content

4. **Create the Confluence page** using the Atlassian MCP tools:
   - Space: from user input
   - Title: from the markdown heading
   - Parent page: from user input (if provided)
   - Body: convert the markdown content to Confluence-compatible format

5. **Store the page metadata** as a comment at the top of the markdown file so future publishes know where to update:

```markdown
<!-- confluence-space: ENG -->
<!-- confluence-title: RFC: Migrate Auth to OAuth 2.0 -->
<!-- confluence-page-id: 12345678 -->
```

6. **Report the result** — confirm the page was created and share the URL if available.

### When triggered by another skill (e.g., write-rfc)

The calling skill provides the file path. This skill handles everything else — MCP check, user questions, page creation.

### When used standalone

If invoked directly:
1. Ask the user which markdown file to publish, or accept a file path argument
2. Check if the file already has confluence metadata (`<!-- confluence-page-id:`)
   - If yes, skip to Mode 2 (Publish)
   - If no, run the Setup flow above

## Mode 2: Publish

Use this mode to update an already-connected Confluence page.

### Steps

1. **Read the confluence metadata** from the markdown file comments (`confluence-space`, `confluence-title`, `confluence-page-id`)

2. **Read the markdown content** (excluding the metadata comments)

3. **Update the Confluence page** using the Atlassian MCP tools with the stored page ID

4. **Handle the result**:
   - **Success**: Briefly confirm the page was updated (one line)
   - **Failure**: Show the error and suggest fixes:
     - Auth error → re-run MCP setup or re-authenticate
     - Page not found → page may have been deleted, switch to Setup mode
     - Permission error → check space permissions

## Integration Protocol

When another skill (like write-rfc) uses this skill as a publisher:

1. The calling skill invokes `md-to-confluence` with the file path
2. On **first invocation**: this skill runs Setup (checks MCP, asks space/parent, creates page, stores metadata)
3. On **subsequent invocations**: this skill runs Publish only (detects existing metadata via `<!-- confluence-page-id:`)

The calling skill doesn't need to track state — this skill auto-detects whether setup has been done by checking for confluence metadata in the file.

## Important

- **Never modify the document content** — only inject/update confluence metadata comments at the top of the file
- **Keep output minimal** during publish — the user is focused on writing, not on publish logs
- **Don't block the writing flow** — if publish fails, report it briefly and let the user continue writing
- **Page ID is the source of truth** — always use `confluence-page-id` for updates, not title matching
