# Intent

## Goal
A support coordinator needs to preview the tickets in a CSV file before anyone approves a later import. In a future round, Claude will add a pure preview function that takes CSV text and returns either every ticket it parsed, in input order, or a single error code. The preview never changes data, so the coordinator can see exactly what would be imported, or why the file was rejected, at no risk. This round only produces this intent document. No feature code is written now.

## Scope
Claude must ask the user for approval before implementing anything, and again before expanding scope. When approved, the only files that may be added are `src/import-tickets.mjs` and `tests/import-tickets.test.mjs`. Excluded work: any UI, database, network call, file write, or new package dependency. Claude must not edit `src/tickets.mjs`, anything under `.github/`, or `CLAUDE.md`. Importing tickets for real, and approving an import, are out of scope; this is preview only. During this round, the only file Claude may change is `intent.md`.

## Constraints
The existing `src/tickets.mjs` is the stable ticket representation and must stay untouched. The preview has no side effects: it reads the CSV string it is given and returns a value. It does not write files, call the network, or mutate inputs. The header must contain a `title` column and may contain a `priority` column; any other, missing or duplicate column name is rejected with `unknown-column`, as is a malformed row or an empty CSV with no header. Values are trimmed of surrounding whitespace. An empty title is `empty-title`. Priorities are `low`, `normal` and `urgent`; a missing or blank priority becomes `normal`, and anything else is `invalid-priority`. At most 100 data rows are accepted, and this is checked before any row value is validated, so 101 rows returns `row-limit` even if the rows are otherwise invalid. CSV quoting must be respected, for example a quoted title that contains a comma. Success is `{"tickets":[{"title":"...","priority":"normal"}]}` in input order, and failure is `{"error":"error-code"}`. Partial success is never returned.

## Plan
1. Stop and ask the user to approve implementation. Do not start without it.
2. Read `src/tickets.mjs` and its tests to learn the existing conventions, without changing them.
3. Write failing tests first in `tests/import-tickets.test.mjs`, one per acceptance case below plus trimming, a duplicate column and a malformed row.
4. Implement a small CSV parser with quote handling in `src/import-tickets.mjs`, then the checks in this order: header columns, row count limit, then per-row title and priority validation.
5. Return the success or failure shape exactly, with no partial results.
6. Run the verification commands and report the results. Ask before touching any other file.

## Acceptance
The JSON contract below is the objective checklist. Positive cases cover an explicit priority, a defaulted priority and a quoted title containing a comma. Negative cases cover an empty title, an unknown column, an invalid priority and too many rows. Extra cases cover a duplicate column, a malformed row, an empty CSV and whitespace trimming. In the too-many-rows case, `repeat` of 101 means 101 copies of the single data row. Every listed outcome must be observable from the returned value alone, and no case may change any data.

```json
{
  "schema": 1,
  "change": "ticket-csv-preview",
  "allowed_changes": ["src/import-tickets.mjs", "tests/import-tickets.test.mjs"],
  "protected_paths": ["src/tickets.mjs", ".github/", "CLAUDE.md"],
  "non_goals": ["ui", "database", "network"],
  "requirements": {
    "columns": ["title", "priority"],
    "default_priority": "normal",
    "valid_priorities": ["low", "normal", "urgent"],
    "reject_unknown_columns": true,
    "reject_empty_title": true,
    "max_rows": 100,
    "side_effects": "preview-only"
  },
  "verification": ["npm test", "git diff --check"],
  "approval": "ask-before-implementation",
  "acceptance_cases": [
    {
      "csv": "title,priority\nPrinter issue,urgent\n",
      "expected": {"tickets":[{"title":"Printer issue","priority":"urgent"}]}
    },
    {
      "csv": "title\nReset password\n",
      "expected": {"tickets":[{"title":"Reset password","priority":"normal"}]}
    },
    {
      "csv": "title,priority\n\"Printer, floor 2\",low\n",
      "expected": {"tickets":[{"title":"Printer, floor 2","priority":"low"}]}
    },
    {
      "csv": "title,priority\n  ,low\n",
      "expected": {"error":"empty-title"}
    },
    {
      "csv": "title,owner\nPrinter issue,sam\n",
      "expected": {"error":"unknown-column"}
    },
    {
      "csv": "title,priority\nPrinter issue,high\n",
      "expected": {"error":"invalid-priority"}
    },
    {
      "csv": "title,priority\nPrinter issue,low\n",
      "repeat": 101,
      "expected": {"error":"row-limit"}
    },
    {
      "csv": "title,priority\n  Login bug  , urgent \n",
      "expected": {"tickets":[{"title":"Login bug","priority":"urgent"}]}
    },
    {
      "csv": "title,title\nPrinter issue,low\n",
      "expected": {"error":"unknown-column"}
    },
    {
      "csv": "",
      "expected": {"error":"unknown-column"}
    }
  ]
}
```

## Verification
After implementation, run `npm test` and `git diff --check`; both must pass with no whitespace errors. Also run `git status` and confirm that only `src/import-tickets.mjs` and `tests/import-tickets.test.mjs` changed, and that `src/tickets.mjs`, `.github/` and `CLAUDE.md` are untouched. Compare each acceptance case above with the actual function output. Ask the user for approval before implementing, before expanding scope, and before any commit, push or PR. Do not claim success until the commands have run and their output has been shown.
