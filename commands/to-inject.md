---
description: Convert constructor parameter DI to the inject() function form (private foo: Foo → private foo = inject(Foo)).
argument-hint: <file-or-folder>
---

Run the `to-inject` operation from the `angular-modernize` skill on: $ARGUMENTS

If $ARGUMENTS is empty, ask the user which file or folder. Do not run on the whole project without a target.

Read **before editing** and follow precisely:
- `skills/angular-modernize/references/shared-conventions.md`
- `skills/angular-modernize/references/to-inject.md`

Apply to every `.ts` file in the target. Convert each constructor parameter that has an access modifier (`private`/`protected`/`public`/`readonly`) into a class field initialized with `inject(...)`. Preserve modifiers, order, and decorator semantics (`@Optional`, `@SkipSelf`, etc. → options object). Empty out the constructor signature; remove the constructor if its body becomes empty AND the class has no `super()` call.

Report changed files and any `TODO(ng-migrate): to-inject: ...` lines. Ask before running verification.
