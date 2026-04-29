# Operation: take-until-destroyed

Add `takeUntilDestroyed(this.destroyRef)` to every `.subscribe()` in the file so subscriptions clean up automatically.

Requires Angular **v16+**. Imported from `@angular/core/rxjs-interop`.

## Rules

For every `.subscribe(...)` call in the file:

1. **Already has `takeUntilDestroyed(this.destroyRef)` in its `pipe()`** → do nothing.
2. **Has a `pipe(...)` but not `takeUntilDestroyed`** → add `takeUntilDestroyed(this.destroyRef)` as the **last** operator inside that `pipe()`.
3. **No `pipe(...)` before `.subscribe()`** → insert `.pipe(takeUntilDestroyed(this.destroyRef))` immediately before `.subscribe()`.

Apply to **all** occurrences in the file. Do not stop after the first match.

## DestroyRef injection

If the class does not already have `DestroyRef` injected, add:

```ts
private destroyRef = inject(DestroyRef);
```

Placement: append to the existing `inject(...)` block (see `to-inject.md` § 5). Concretely:

1. If the class already has `inject(...)` fields, add `destroyRef` as the last field in that block.
2. If the class has no `inject(...)` fields yet, place `destroyRef` directly above the constructor (or where the constructor would be if absent), after all signals / computed / getters / setters.
3. Never interleave with plain fields, signals, computed, getters, or setters.

Do not duplicate if `destroyRef` already exists under a different name (e.g. `_destroyRef`); use whatever name is already there in the `pipe(takeUntilDestroyed(this.<name>))` call.

## Imports

Ensure these are present (add if missing, do not duplicate):

```ts
import { DestroyRef, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
```

## Do NOT

- Refactor logic, rename variables, or change formatting unnecessarily.
- Touch `.subscribe()` calls on non-RxJS sources (event emitters, custom subscribe methods).
- Combine this operation with `.subscribe({...})` observer-object conversion — that's a separate operation (`subscribe-object-form`).

## Edge cases

- **Subscribe inside a constructor / `ngOnInit` / arrow function:** still applies — `this.destroyRef` only requires being inside a class instance method.
- **Subscribe inside a static method or non-class context:** leave a `// TODO(ng-migrate): take-until-destroyed: not in class instance context — manual review needed` and skip.
- **Subscribe chained off a method call, e.g. `this.svc.foo().subscribe(...)`:** still wrap with `.pipe(takeUntilDestroyed(this.destroyRef)).subscribe(...)`.

## Final assertion

After the pass: zero `.subscribe(` occurrences in the file should be without an upstream `takeUntilDestroyed(this.destroyRef)` in their pipe (or a TODO comment).
