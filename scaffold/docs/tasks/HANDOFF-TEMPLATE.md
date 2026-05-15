# Slice handoff: [task short title] — Slice N

| Field | Value |
|--------|--------|
| **Task file** | [`open/YYYY-MM-DD-slug.md`](open/YYYY-MM-DD-slug.md) |
| **Slice** | N — [one-line name] |
| **Completed** | YYYY-MM-DD |

## RED

- What failing test or behaviour was added (file paths).
- What was expected to fail and why (if applicable).

## GREEN

- What changed to make the slice pass (paths; “none” if behaviour already existed).
- Config or test-only changes called out explicitly.

## DESIGN REVIEW (no code changes)

- List any issues found against project patterns:
  - misplaced responsibilities
  - unnecessary abstractions / over-engineering
  - style or convention deviations
  - overly defensive code
- If none, explicitly state: `No design issues found`.

## REFACTOR

- What structural improvements were made without behaviour changes.
- If no refactor was needed, explicitly state: `No refactor needed`.

## VERIFY

```bash
# Example: commands to re-run the slice’s tests
./gradlew …
```

## Notes

- Strict TDD: did the test fail first, or was GREEN immediate?
- Blockers, follow-ups, or things the next slice must not assume.

## Next slice

- One sentence: what the next RED test should target (link to task tracker in task file).
