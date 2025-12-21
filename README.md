# Code Review Dataset Export (GitHub MCP)

This repo contains datasets of GitHub PR review comments pulled via the GitHub MCP server. Use the MCP tools in Cursor (or any MCP host) to fetch PR review comments, review bodies, and issue comments, then normalize them into JSON/CSV.

## Prereqs
- PAT: fine-grained, read-only to the target repos (or minimal classic `repo` if required).
- GitHub MCP server configured in `.cursor/mcp.json`:
  ```json
  "github": {
    "command": "docker",
    "args": [
      "run", "-i", "--rm",
      "-e", "GITHUB_PERSONAL_ACCESS_TOKEN",
      "-e", "GITHUB_READ_ONLY=1",
      "ghcr.io/github/github-mcp-server:latest"
    ],
    "env": {}
  }
  ```
- Export the token before launching Cursor/MCP:
  ```bash
  export GITHUB_PERSONAL_ACCESS_TOKEN=ghp_xxx
  export GITHUB_READ_ONLY=1
  ```

## Workflow (summary)
1) List PRs: `pull_request_read:list_pull_requests` (owner/repo/state/perPage).
2) For each PR:
   - Review comments: `pull_request_read:get_review_comments`
   - Review bodies: `pull_request_read:get_reviews`
   - Issue comments: `issue_read:get_comments` (PR issue number)
3) For each code comment: fetch exact lines from the comment commit using `original_commit_id`/`commit_id`, `path`, and `line`. Extract a 4-line window starting at `line`.
4) Normalize to JSON/CSV; filter out `cursor[bot]` entries.

## Current dataset
- `datasets/alix-instructions-registry-PRs.json`: review comments for `meetalix/alix-instructions-registry`, filtered to exclude `cursor[bot]`, with context snippets.

See `docs/ingestion-guide.md` for detailed steps and the context-extraction script pattern.

