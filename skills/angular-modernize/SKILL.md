---
name: angular-modernize
description: Migrate Angular code from legacy patterns to modern (v17–v18+) idioms. Six mechanical migrations — subscribe object form, constructor DI to inject(), takeUntilDestroyed cleanup, signals (TS + paired HTML, including @Input/@Output → input()/output() and two-way binding splits), and new template control flow (@if / @for / @switch). TRIGGER when the user asks to migrate, modernize, refactor, or convert Angular code, mentions any of "signals", "takeUntilDestroyed", "destroyRef", "@if", "@for", "control flow", "subscribe object form", "inject", "input()", "output()", "new angular template", or asks to run any of these operations on a specific file or folder. SKIP for non-Angular code or when no target file/folder is named.
---

# Angular Modernize

Mechanical, behavior-preserving migrations from legacy Angular to modern Angular (v17–v18+, Signals-friendly).

## Hard rules

1. **Never run on "everything" or the whole project without a named target.** Require the user to name a file, list of files, or folder. If they don't, ask.
2. **Behavior-preserving only.** Do not refactor logic, rename variables, or change formatting beyond what each operation requires.
3. **Paired-file rule for signals:** when running signals on a `.ts` component file, also migrate its paired template (same basename `.html`, or whatever `templateUrl` resolves to, or the inline `template:` string) so signal reads in the template get `()` and two-way bindings get split. If a paired template isn't found, note it and continue with TS only.
4. **Folder targets** apply each operation to every applicable file in the folder (recursively). `.ts` operations on `.ts` files; `.html` operations on `.html` files; signals on `.ts` + paired `.html`. Skip `*.spec.ts` unless the user explicitly opts in.
5. **Unsafe transformations** are not silently skipped. Leave a comment in this exact format:
   `// TODO(ng-migrate): <operation>: <reason>` (TS) or `<!-- TODO(ng-migrate): <operation>: <reason> -->` (HTML).
6. **Coverage is mandatory.** Each operation has a coverage check section in its reference file (especially `ts-to-signals.md`). Run that check before reporting the operation done. Missing fields are bugs — go back and fix them, don't ship a half-migrated file.
7. **Verification:** after each operation finishes, ask the user whether to run a verification command (typecheck/build). Do not run it without consent. Detect the command from `package.json` scripts when possible (`typecheck`, `build`, `lint`); otherwise ask the user what to run.
8. **Angular version:** detect from `package.json` `dependencies.@angular/core` if present. Warn before running operations that require a newer version than detected:
   - `to-inject` works on v14+, recommended v16+
   - `take-until-destroyed` requires v16+
   - `signals` (including `input()` / `output()`) recommended v17.1+ for `input()` / `output()`, v16+ for plain `signal()`
   - `template-control-flow` requires v17+

## Slash command

There is exactly one slash command in this plugin:

- `/modernize <file-or-folder>` — runs all six operations in canonical order on the target.

For anything else, **infer from natural language and context**. The user names a target and describes what they want; pick operations from the routing table below.

## Routing — pick the operation(s) from the user's phrasing

Match against these triggers, then read the matching reference file(s) and follow them precisely. Multiple operations may apply when the user says "modernize" / "full migration".

| User phrasing                                                                                                                | Operation(s)                  | Reference file                                                              |
| ---------------------------------------------------------------------------------------------------------------------------- | ----------------------------- | --------------------------------------------------------------------------- |
| "subscribe object form", "modern subscribe", "next/error object", "deprecated subscribe", "fix subscribe callbacks"          | `subscribe-object-form`       | `references/subscribe-object-form.md`                                       |
| "to inject", "constructor to inject", "inject() function", "modern DI", "remove constructor injection", "rewrite DI"         | `to-inject`                   | `references/to-inject.md`                                                   |
| "takeUntilDestroyed", "destroyRef", "subscribe cleanup", "fix subscribes", "auto-unsubscribe"                                | `take-until-destroyed`        | `references/take-until-destroyed.md`                                        |
| "to signals", "convert to signals", "migrate to signals", "signal-ify", "@Input to input()", "@Output to output()"           | `signals` (TS + paired HTML)  | `references/ts-to-signals.md` + `references/template-to-signals.md`         |
| "new template", "control flow", "*ngIf to @if", "@for", "@switch", "new angular template syntax"                             | `template-control-flow`       | `references/template-control-flow.md`                                       |
| "modernize", "migrate to new angular", "full migration", "new angular flow/look/layout/pattern", or `/modernize` is invoked  | **all six**, canonical order  | run all references                                                          |

## Canonical order when running multiple operations

Regardless of how the user listed operations, run them in this order — each on its applicable file types only:

1. `subscribe-object-form` (`*.ts`) — pure syntactic, safest first.
2. `to-inject` (`*.ts`) — restructures constructor; doesn't touch logic. Inject block lands at the constructor's position (see `to-inject.md` § 5).
3. `take-until-destroyed` (`*.ts`) — adds `destroyRef = inject(DestroyRef)` to the inject block.
4. `signals` (`*.ts` + paired `*.html`) — biggest change. Includes `@Input()` → `input()`, `@Output()` → `output()`, with FormGroup/FormControl `@Input`s left as TODO. Coverage check runs at the end. The paired template pass also splits two-way `[(prop)]="x"` bindings.
5. `template-control-flow` (`*.html`) — `*ngIf`/`*ngFor`/`[ngSwitch]` → new syntax. Runs last so it operates on the already-updated template (signal reads + two-way splits).

## Shared conventions

Always follow `references/shared-conventions.md` for:
- TODO comment format
- Injection detection (`inject(...)` vs constructor DI)
- Paired-file resolution (`foo.component.ts` ↔ `foo.component.html`)
- Imports cleanup (remove unused, add missing from `@angular/core`)

## Workflow per invocation

1. Confirm the target. If the user named files/folders, use those. If not, ask.
2. Detect Angular version from `package.json`. Warn if a requested operation requires newer.
3. Determine which operation(s) to run from the routing table (or all six if `/modernize`).
4. Read the matching reference file(s) **before** editing — they contain the precise rules and edge cases.
5. Apply edits using the `Edit` or `Write` tools. Do not paste output as a chat message.
6. Run the operation's coverage check before declaring it done.
7. After each operation, list changed files and any `TODO(ng-migrate)` comments left.
8. Ask the user whether to run a verification command (auto-detected or asked for).
