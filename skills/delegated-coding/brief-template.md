# Brief template

Copy this template for every dispatch to `code-writer` or `test-writer`. Fill
all seven fields. Do not dispatch a brief with an empty field.

## Template

```
GOAL
One sentence. What exists after this task that does not exist now.

FILES
Exact paths to create or edit. One per line. Mark each one create or edit.

SIGNATURES
The exact function, type, class, and interface signatures to produce.
Copy them as code, not as prose.

PATTERN TO COPY
A file path with a line range, or a pasted snippet. Name what to copy from it:
structure, error handling, naming, import order.

CONSTRAINTS
Rules that bound the work. Cover new dependencies, style, formatting,
file scope, and anything not to touch.

ACCEPTANCE
Shell commands that must exit 0. One per line. Run from the repository root.

RETURN
FILES CHANGED, COMMANDS RUN with exit codes, NOTES.
```

## Filled example

This example delegates one small function and its test.

```
GOAL
Add a parseRetryAfter helper that turns an HTTP Retry-After header value into
a delay in milliseconds.

FILES
create  src/http/parseRetryAfter.ts
create  src/http/parseRetryAfter.test.ts

SIGNATURES
export function parseRetryAfter(
  header: string | null,
  now: Date = new Date()
): number | null;

Return the delay in milliseconds. Return null when the header is null, empty,
or unparseable. Return 0 when the computed delay is negative.

PATTERN TO COPY
src/http/parseContentRange.ts lines 1 to 44.
Copy the import order, the named export style, the early-return shape, and the
JSDoc block above the function.
Copy the test layout from src/http/parseContentRange.test.ts lines 1 to 30.

CONSTRAINTS
No new dependencies.
TypeScript strict mode is on. Do not use `any`.
Handle both Retry-After forms: a number of seconds, and an HTTP date.
Do not edit any file outside the two listed above.
Do not export anything else from the new file.

ACCEPTANCE
npx tsc --noEmit
npx vitest run src/http/parseRetryAfter.test.ts
npx eslint src/http/parseRetryAfter.ts src/http/parseRetryAfter.test.ts

RETURN
FILES CHANGED, COMMANDS RUN with exit codes, NOTES.
```

## Notes on filling the fields

1. Write the signature yourself. Never ask the subagent to choose a signature.
2. Give a line range for the pattern, not a file name alone. A whole file is
   too much to copy from.
3. Write acceptance commands you have already run in this repository. A command
   that does not exist wastes the dispatch.
4. Name the files not to touch when the repository has files that look similar.
5. Keep one brief to one unit of work. Two unrelated functions are two briefs.
