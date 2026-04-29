# Operation: to-inject (constructor DI → `inject()` function)

Convert legacy constructor parameter injection to the `inject()` function form. Requires Angular **v14+** (recommended v16+ for full support including `inject` in field initializers without manual workarounds).

## Goal

Before:
```ts
constructor(
  private schoolAdminService: SchoolAdminService,
  private confirmationService: ConfirmationService,
  private messageService: MessageService,
  private fb: UntypedFormBuilder,
  private blockUIService: BlockUIService,
  private technicalContactService: TechnicalContactService,
  private addContractComponent: AddContractComponent,
) {
  super();
  this.initForm();
}
```

After:
```ts
private schoolAdminService = inject(SchoolAdminService);
private confirmationService = inject(ConfirmationService);
private messageService = inject(MessageService);
private fb = inject(UntypedFormBuilder);
private blockUIService = inject(BlockUIService);
private technicalContactService = inject(TechnicalContactService);
private addContractComponent = inject(AddContractComponent);

constructor() {
  super();
  this.initForm();
}
```

## Rules

### 1. Eligible parameters

A constructor parameter is eligible **if and only if** it has an access modifier (`private`, `protected`, `public`, or `readonly`) — those are the parameters TypeScript turns into class fields. Plain parameters (no modifier) are local to the constructor body and stay in place.

```ts
// eligible (becomes a field):
constructor(private foo: Foo) {}

// NOT eligible (local-only):
constructor(foo: Foo) {}
```

### 2. Conversion

For each eligible parameter `<modifier> <name>: <Type>`:

- Create a class field: `<modifier> <name> = inject(<Type>);`
- Place it where existing `inject(...)` fields live, or directly above the constructor.
- Preserve all modifiers exactly (`private`, `protected`, `public`, `readonly`, combinations like `private readonly`).
- Remove the parameter from the constructor signature.

If the field already exists with the same name (rare collision), skip and add:
`// TODO(ng-migrate): to-inject: name collision for <name> — manual review`

### 3. Decorators on parameters

Convert each common DI decorator to the `inject()` option object:

| Decorator                       | Conversion                                                |
| ------------------------------- | --------------------------------------------------------- |
| `@Inject(TOKEN) private x: T`   | `private x = inject(TOKEN);`                              |
| `@Optional() private x: T`      | `private x = inject(<Type>, { optional: true });`         |
| `@Self() private x: T`          | `private x = inject(<Type>, { self: true });`             |
| `@SkipSelf() private x: T`      | `private x = inject(<Type>, { skipSelf: true });`         |
| `@Host() private x: T`          | `private x = inject(<Type>, { host: true });`             |

Combine multiple decorators by merging into one options object:
```ts
@Optional() @SkipSelf() private x: T
// → private x = inject<T | null>(<Type>, { optional: true, skipSelf: true });
```

For `@Inject(TOKEN)` where `TOKEN` is an `InjectionToken<T>`, the type is inferred — drop the type annotation in the new form unless it differs from the token's type.

If a decorator is unfamiliar (e.g. a custom DI decorator), leave the parameter as-is and add:
`// TODO(ng-migrate): to-inject: unknown decorator <Decorator> — manual review`

### 4. Constructor body handling

After moving all eligible parameters out:

- If the constructor body is **non-empty** (has any statement, including `super()`), keep the constructor with an empty parameter list:
  ```ts
  constructor() {
    super();
    this.initForm();
  }
  ```
- If the constructor body is **empty AND the class does not extend another class**, remove the constructor entirely.
- If the constructor body is **empty BUT the class extends another class** with a constructor that requires `super()`, keep an empty constructor with `super()`:
  ```ts
  constructor() {
    super();
  }
  ```
  (Many base classes have a no-arg constructor and don't strictly need `super()`, but it's safe to keep it.)
- If `super(...)` is called with arguments that came from the constructor parameters, leave those parameters in the signature and do not convert them. Add:
  `// TODO(ng-migrate): to-inject: parameter <name> is forwarded to super(...) — manual review`

### 5. Field placement

Order:
1. If the class already has `inject(...)` fields, append the new ones immediately after the last existing one.
2. Otherwise, place the block of new `inject(...)` fields just above the constructor (or, if the constructor is being removed, just above the first method).
3. Preserve relative order of injected services (same order as in the original constructor parameter list).
4. Do not interleave with non-inject fields. The injected fields form one contiguous block.

### 6. Imports

Ensure `inject` is imported from `@angular/core`:
```ts
import { inject } from '@angular/core';
```
Add it if missing. Do not remove existing imports of injected types — they're still used as the argument to `inject(...)`.

### 7. Do not refactor

- Do not rename injected services.
- Do not change visibility (`private` stays `private`, etc.).
- Do not reorder the existing class members beyond the placement rule for new inject fields.
- Do not touch usages — `this.fooService.x()` continues to work identically.

## Edge cases

- **Manual injector usage** (`Injector.get(...)`, `this.injector.get(...)`): not in scope; leave as-is.
- **Field-initializer ordering**: if a non-inject field initializer references `this.<injectedService>` (e.g. `data = this.svc.getInitial();`), keep that — `inject()` works in field initializers in v14+, and the order of field declarations determines initialization order. If a non-inject field appears before the inject fields after migration, swap so all `inject(...)` fields come first.
- **`@Injectable({ providedIn: 'root' })` services with constructor DI**: same migration applies.
- **`@Component`, `@Directive`, `@Pipe` classes**: same migration applies.

## Validation checklist

- Every former parameter-property is now a class field with `inject(...)`.
- Constructor signature is empty `constructor()` or removed (per rules above).
- All access modifiers preserved.
- `inject` import present.
- No call sites changed (`this.foo` references still resolve).
- No injected service was lost or renamed.
