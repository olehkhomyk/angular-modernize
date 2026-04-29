---
description: Migrate Angular component(s) to signals (TS class fields + paired template). Argument is a file or folder path.
argument-hint: <file-or-folder>
---

Run the `signals` operation from the `angular-modernize` skill on the target: $ARGUMENTS

If $ARGUMENTS is empty, ask the user which file or folder to migrate. Do not run on the whole project without an explicit target.

Read these reference files from the skill **before editing** and follow them precisely:
- `skills/angular-modernize/references/shared-conventions.md`
- `skills/angular-modernize/references/ts-to-signals.md`
- `skills/angular-modernize/references/template-to-signals.md`

Execution:
1. Apply `ts-to-signals` rules to every `.ts` file in the target. Be aggressive about coverage — every plain stateful field must become a signal unless it matches the explicit skip list (decorators, signal-form, inject(), readonly/static, observables, method-like fields).
2. For each migrated `.ts`, locate its paired `.html` (or inline template) and apply `template-to-signals` rules so signal reads use `()`.
3. Run the **mandatory coverage check** at the end of `ts-to-signals.md` before reporting done. If any plain stateful field remains non-signal, go back and fix it.
4. Report: list of changed files, count of signals introduced, any `TODO(ng-migrate)` lines inserted (with file:line).
5. Ask the user before running any verification command (typecheck/build/lint).
