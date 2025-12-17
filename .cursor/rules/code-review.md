# Code Review Heuristics (field notes)

Use these as a checklist to mirror what reviewers tend to flag.

## Backend / Schema / Services
- Keep models minimal: avoid extra columns unless there is a clear, referenced requirement (reviewers push back on denormalization or duplicated summary fields when data already lives in a relation).
- Avoid speculative perf: joins/aggregations are acceptable; justify any “summary on parent” fields with concrete UI/API needs.
- Be explicit with enums: prefer exhaustive `switch`/guards so new enum members surface compile-time errors.
- Use generated types, not deprecated model types.
- Prefer server-generated timestamps; don’t accept `createdAt` from inputs.
- For identifiers/counters (e.g., attemptNumber), justify with concrete idempotency/ordering needs; otherwise prefer counting/sorting.
- Use shared retry/backoff helpers instead of ad‑hoc loops when available.

## Migration / Box scripts
- When reusing existing destination folders, ensure all mapped references (beneficiary/executor/contact/user) are re-pointed or explicitly cleared; don’t skip contacts silently.
- Log and handle missing mappings; don’t leave stale IDs in DB.
- Remove unused parameters to avoid drift between intent and code.

## Frontend / Feature flags
- When gating routes by flags, ensure behavior for both enabled/disabled paths is explicit (avoid component mounting bugs when flag flips).
- Prefer redirect or clear fallback UI over null renders where possible.

## E2E / Playwright patterns
- Locators: prefer stable data-test ids; avoid brittle text/role-only selectors where a test id is feasible. Ask for test ids if missing.
- Helpers should not silently change page state (e.g., hidden searches inside visibility checks). Keep actions explicit.
- Timeouts: use shared constants, avoid magic numbers.
- Waiting: avoid arbitrary sleeps; use `waitForTimeout` or expect-based waits tied to a condition.
- Naming: test names should be human-readable (avoid JSON.stringify blobs); method/variable names should convey intent.
- Comments/logs: remove commented-out code and stray loggers; keep comments in English.
- Data generators: prefer simple, explicit methods per case/status rather than branching mega-generators; keep generated data deterministic/unique when needed (e.g., titles).
- Translations: avoid non-English strings in test code.
- Formatting: run lint/formatter; fix stray blank lines/indentation.

## PR hygiene
- Follow naming conventions for PRs/branches.
- Document rationale when adding fields, flags, or retry logic so reviewers see the requirement link.

