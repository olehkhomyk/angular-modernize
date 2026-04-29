---
description: Add takeUntilDestroyed(this.destroyRef) to every .subscribe() and inject DestroyRef if missing.
argument-hint: <file-or-folder>
---

Run the `take-until-destroyed` operation from the `angular-modernize` skill on: $ARGUMENTS

If $ARGUMENTS is empty, ask the user which file or folder. Do not run on the whole project without a target.

Read **before editing** and follow precisely:
- `skills/angular-modernize/references/shared-conventions.md`
- `skills/angular-modernize/references/take-until-destroyed.md`

Apply to every `.ts` file in the target. After the pass, every `.subscribe(` in the file must have an upstream `takeUntilDestroyed(this.destroyRef)` in its `pipe()` (or a TODO comment explaining why not).

Report changed files and any `TODO(ng-migrate): take-until-destroyed: ...` lines. Ask before running verification.
