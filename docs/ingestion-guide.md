# Ingestion Guide: GitHub PR Review Comments (MCP)

## Goals
- Pull review comments, review bodies, and PR issue comments from GitHub repos.
- Attach minimal, accurate code context (4 lines starting at the commented line).
- Exclude bot noise (`cursor[bot]`).

## Prereqs
- Fine-grained PAT, read-only to target repos (or minimal classic `repo` if FG not allowed).
- Env:
  ```bash
  export GITHUB_PERSONAL_ACCESS_TOKEN=ghp_xxx
  export GITHUB_READ_ONLY=1
  ```
- MCP config (Cursor) includes the GitHub MCP server:
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

## Data you need per PR
- Review comments (code): `pull_request_read:get_review_comments`
- Review bodies: `pull_request_read:get_reviews`
- Issue comments (top-level on PR): `issue_read:get_comments`
- (Optional) Files/commits for alignment: `pull_request_read:get_files`, `pull_request_read:get_commits`

## Context extraction (must be precise)
For each review comment that has line/commit info:
1. Read fields: `path`, `line` (or `original_line`), `commit_id` and `original_commit_id`.
2. Choose the commit for context:
   - Prefer `original_commit_id` if present; else `commit_id`.
3. Extract exactly 4 lines starting at the commented line from that commit:
   ```bash
   git show <commit>:<path> | sed -n '<line>,<line_plus_3>p'
   ```
4. Store that snippet as `context`. Do NOT include unrelated surrounding code.

File-level comments (no line) can have empty context; review bodies and issue comments remain unchanged.

## Filtering
- Drop any entries where `author == "cursor[bot]"`.
- Keep author, body, created_at, path, line, commit_id/original_commit_id, and context.

## JSON schema (example)
```json
{
  "repo": "meetalix/alix-instructions-registry",
  "pr": 5,
  "comment_type": "review_comment",   // or review_body, issue_comment
  "author": "nasekajlo",
  "path": "frontend/src/pages/JobsList.tsx",
  "line": 122,
  "body": "what if jobs.length === 0? ...",
  "created_at": "2025-12-16T21:13:12Z",
  "resolved": false,
  "outdated": false,
  "commit_id": "e74f19423c6c8a27f31a1e8a36011677d6a9177b",
  "original_commit_id": "8f5d5c9e05beacd7bea2ab4dd0236fdddbfbeeb4",
  "context": "            Next\n          </Button>\n        </Box>\n      )}\n"
}
```

## Example MCP calls (read-only)
- List PRs: `pull_request_read:list_pull_requests` with `{ owner, repo, state: "all", perPage: 50 }`
- Review comments: `pull_request_read:get_review_comments` with `{ owner, repo, pullNumber, perPage: 100 }`
- Review bodies: `pull_request_read:get_reviews`
- Issue comments: `issue_read:get_comments` with `{ issue_number: <pr_number> }`

## Export targets
- JSON: `datasets/<repo>-PRs.json`
- (Optional) CSV: same fields flattened; keep context escaped.

## Notes
- Always use the comment’s commit for context, not the current head.
- Keep snippets minimal (4 lines starting at the commented line).
- Stay read-only: do not use write MCP methods; keep the PAT scoped read-only.

