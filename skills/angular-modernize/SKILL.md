---
name: angular-modernize
description: Migrate Angular code from legacy patterns to modern (v17–v18+) idioms. Six mechanical migrations — subscribe object form, constructor DI to inject(), takeUntilDestroyed cleanup, signals (TS + paired HTML, including @Input/@Output → input()/output() and two-way binding splits), and new template control flow (@if / @for / @switch). TRIGGER when the user asks to migrate, modernize, refactor, or convert Angular code, mentions any of "signals", "takeUntilDestroyed", "destroyRef", "@if", "@for", "control flow", "subscribe object form", "inject", "input()", "output()", "new angular template", or asks to run any of these operations on a specific file or folder. SKIP for non-Angular code or when no target file/folder is named.
---

# Angular Modernize

Mechanical, behavior-preserving migrations from legacy Angular to modern Angular (v17–v18+, Signals).

## Hard rules

1. **Always require a target.** File, list of files, or folder. No target → ask. Never run on the whole project / cwd.
2. **Behavior-preserving only.** No logic refactors, renames, or formatting churn.
3. **One pass per file (token-efficient).** For each target file: `Read` once → apply all in-scope transformations to the in-memory string → `Write` once. Do NOT chain multiple `Edit`s on the same file. Fall back to `Edit`s only when a file is too large for a single `Write`.
4. **No mid-run summaries or verification prompts.** Single combined summary at the end + single verification prompt. Don't narrate per-operation progress.
5. **Paired-file rule (signals only):** when migrating a `.ts` to signals, also migrate the paired template (same basename `.html`, or `templateUrl`, or inline `template:`) so signal reads get `()` and `[(prop)]` two-way bindings split. If no paired template, note and continue.
6. **Folder targets** recurse. `.ts` ops on `.ts`, `.html` ops on `.html`. Skip `*.spec.ts` unless opted in.
7. **Unsafe → TODO**, never silent skip. Format: `// TODO(ng-migrate): <op>: <reason>` (TS) or `<!-- ... -->` (HTML).
8. **Coverage check** for `signals-ts` is mandatory — every plain stateful field must become a signal unless it matches the explicit skip list. See `references/ts-to-signals.md` § Coverage check.
9. **Version detection** from `package.json` `@angular/core`. Skip ops the version is too old for (see `references/shared-conventions.md`).

## Slash command

- `/modernize <file-or-folder>` — runs all six operations on the target in canonical order, one pass per file.

Anything else is natural-language triggered (the user names a target and describes intent).

## Routing — pick op(s) from user phrasing

| Phrasing                                                                                                    | Op(s)                          | Reference                                                                       |
| ----------------------------------------------------------------------------------------------------------- | ------------------------------ | ------------------------------------------------------------------------------- |
| "subscribe object form", "modern subscribe", "next/error object", "fix subscribe callbacks"                 | `subscribe-object-form`        | `references/subscribe-object-form.md`                                           |
| "to inject", "constructor to inject", "inject() function", "modern DI"                                      | `to-inject`                    | `references/to-inject.md`                                                       |
| "takeUntilDestroyed", "destroyRef", "subscribe cleanup", "auto-unsubscribe"                                 | `take-until-destroyed`         | `references/take-until-destroyed.md`                                            |
| "to signals", "convert to signals", "@Input to input()", "@Output to output()"                              | `signals` (TS + paired HTML)   | `references/ts-to-signals.md` + `references/template-to-signals.md`             |
| "new template", "control flow", "*ngIf to @if", "@for", "@switch"                                           | `template-control-flow`        | `references/template-control-flow.md`                                           |
| "modernize", "full migration", "migrate to new angular" / `/modernize` invoked                              | **all six**, canonical order   | run all references                                                              |

## Canonical order (when running multiple ops)

Run on each target file in this fixed order, regardless of how the user listed them:

1. `subscribe-object-form` (`*.ts`)
2. `to-inject` (`*.ts`) — inject block lands at the constructor's position, after signals/computed/getters.
3. `take-until-destroyed` (`*.ts`) — appends `destroyRef` to the inject block.
4. `signals` (`*.ts` + paired `*.html`) — `@Input`/`@Output` migration + two-way template splits + coverage check.
5. `template-control-flow` (`*.html`) — last, so it sees post-signals template.

Apply each op only to its applicable file types.

## Workflow

For each target file:
1. `Read` the file once.
2. Determine which ops apply (from user request + file type + version compatibility).
3. Apply all rules from the relevant reference(s) **in memory**, in canonical order. Read references only once at the start of the run, not per file.
4. `Write` the final content once.

After all files done:
- Produce one combined summary: changed files, counts (signals, injects, subscribes wrapped, control-flow blocks), every `TODO(ng-migrate)` line as `path:line — reason`.
- Ask once: "Run verification? (auto-detected: `<cmd>` from package.json — y/n)". Only the user's "yes" runs it.
