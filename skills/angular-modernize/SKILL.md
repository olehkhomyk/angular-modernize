---
name: angular-modernize
description: Migrate Angular code from legacy patterns to modern (v17–v18+) idioms. Handles five mechanical migrations — takeUntilDestroyed cleanup, modern subscribe observer-object form, signals (TS + paired HTML), and new template control flow (@if / @for / @switch). TRIGGER when the user asks to migrate, modernize, or convert Angular code, mentions any of "signals", "takeUntilDestroyed", "destroyRef", "@if", "@for", "control flow", "subscribe object form", "new angular template", or asks to run any of the five operations on a specific file or folder. SKIP for non-Angular code or when no target file/folder is named.
---

# Angular Modernize

Mechanical, behavior-preserving migrations from legacy Angular to modern Angular (v17–v18+, Signals-friendly).

## Hard rules

1. **Never run on "everything" or the whole project without a named target.** Require the user to name a file, list of files, or folder. If they don't, ask them which.
2. **Behavior-preserving only.** Do not refactor logic, rename variables, or change formatting beyond what each operation requires.
3. **Paired-file rule for signals:** when running `signals` on a `.ts` component file, also migrate its paired template (same basename `.html` next to it) so signal reads in the template get `()`. If a paired template isn't found, note it and continue with TS only.
4. **Folder targets** apply each operation to every applicable file in the folder (recursively). `.ts` operations on `.ts` files; `.html` operations on `.html` files; `signals` on `.ts` + paired `.html`.
5. **Unsafe transformations** are not silently skipped. Leave a comment in the exact format:
   `// TODO(ng-migrate): <operation>: <reason>` (TS) or `<!-- TODO(ng-migrate): <operation>: <reason> -->` (HTML).
6. **Verification:** after each operation finishes, ask the user whether to run a verification command (typecheck/build). Do not run it without consent. Detect the command from `package.json` scripts when possible (`typecheck`, `build`, `lint`); otherwise ask the user what to run.
7. **Angular version:** detect from `package.json` `dependencies.@angular/core` if present. If missing or older than v17, warn the user before running template control-flow migration (it requires Angular 17+).

## Routing — pick the operation(s) from the user's phrasing

Match against these triggers, then read the matching reference file(s) and follow them precisely. Multiple operations may apply when the user says "modernize" / "full migration".

| User phrasing                                                                                                  | Operation(s)                | Reference file                                   |
| -------------------------------------------------------------------------------------------------------------- | --------------------------- | ------------------------------------------------ |
| "takeUntilDestroyed", "destroyRef", "subscribe cleanup", "fix subscribes"                                      | `take-until-destroyed`      | `references/take-until-destroyed.md`             |
| "subscribe object form", "modern subscribe", "next/error object", "deprecated subscribe"                       | `subscribe-object-form`     | `references/subscribe-object-form.md`            |
| "to signals", "convert to signals", "migrate to signals", "signal-ify"                                         | `signals` (TS + paired HTML)| `references/ts-to-signals.md` + `references/template-to-signals.md` |
| "new template", "control flow", "*ngIf to @if", "@for", "@switch", "new angular template syntax"               | `template-control-flow`     | `references/template-control-flow.md`            |
| "modernize", "migrate to new angular", "full migration", "new angular flow/look/layout/pattern"                | **all four**, in order below| run all four references                          |

## Order when running "modernize" (all four)

For each target file (or each file in a target folder), in this order:

1. `subscribe-object-form` — pure syntactic, safest first.
2. `take-until-destroyed` — may inject `DestroyRef` if missing.
3. `signals` — TS first, then paired HTML. Biggest change; needs both files in sync.
4. `template-control-flow` — runs last so it operates on the already-updated template (signal reads with `()`).

Apply each operation to its applicable file types only (e.g. control flow only touches `.html`).

## Shared conventions

Always follow `references/shared-conventions.md` for:
- TODO comment format
- Injection detection (`inject(...)` vs constructor DI)
- Paired-file resolution (`foo.component.ts` ↔ `foo.component.html`)
- Imports cleanup (remove unused, add missing from `@angular/core`)

## Workflow per invocation

1. Confirm the target. If the user named files/folders, use those. If not, ask.
2. Detect Angular version from `package.json` if accessible. Warn if version mismatch is relevant.
3. Determine which operation(s) to run from the routing table above.
4. Read the matching reference file(s) **before** editing — they contain the precise rules and edge cases.
5. Apply edits using the `Edit` or `Write` tools. Do not paste output as a chat message.
6. After each operation, list which files were changed and whether any `TODO(ng-migrate)` comments were left.
7. Ask the user whether to run a verification command (auto-detected or asked for).
