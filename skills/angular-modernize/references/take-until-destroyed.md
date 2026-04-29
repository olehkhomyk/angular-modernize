# take-until-destroyed

Add `takeUntilDestroyed(this.destroyRef)` to every `.subscribe()` in the file. Requires Angular v16+.

## Rules per `.subscribe(...)`

1. Pipe already contains `takeUntilDestroyed(this.destroyRef)` → leave alone.
2. Pipe exists but lacks it → append `takeUntilDestroyed(this.destroyRef)` as the **last** operator.
3. No pipe → insert `.pipe(takeUntilDestroyed(this.destroyRef))` immediately before `.subscribe(...)`.

Apply to ALL `.subscribe(` in the file.

## DestroyRef field

If the class lacks `inject(DestroyRef)` (under any name), add: `private destroyRef = inject(DestroyRef);`

Placement: append to the inject block (see `to-inject.md` § placement). If no inject block exists yet, the field goes directly above the constructor (or where it would be), after signals/computed/getters/setters.

If a `DestroyRef` field already exists under a different name (e.g. `_destroyRef`), reuse that name in the `takeUntilDestroyed(this.<name>)` call.

## Skip

- `.subscribe(...)` outside a class instance context (static method, top-level): leave + TODO: `// TODO(ng-migrate): take-until-destroyed: not in class instance — manual review`.
- Non-RxJS subscribes (EventEmitter, custom).

## Imports

```ts
import { DestroyRef, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
```

## Final check

Zero `.subscribe(` calls in the file without an upstream `takeUntilDestroyed(this.destroyRef)` (or a TODO).
