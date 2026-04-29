---
description: Convert deprecated subscribe(next, error, complete) to observer-object form subscribe({ next, error, complete }).
argument-hint: <file-or-folder>
---

Run the `subscribe-object-form` operation from the `angular-modernize` skill on: $ARGUMENTS

If $ARGUMENTS is empty, ask the user which file or folder. Do not run on the whole project without a target.

Read **before editing** and follow precisely:
- `skills/angular-modernize/references/shared-conventions.md`
- `skills/angular-modernize/references/subscribe-object-form.md`

Apply to every `.ts` file in the target (skip `*.spec.ts` unless the user opts in). Preserve the observable chain, parameter names, formatting, and `this`-binding semantics.

Report changed files and any `TODO(ng-migrate): subscribe-object-form: ...` lines. Ask before running verification.
