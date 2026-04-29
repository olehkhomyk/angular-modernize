# to-inject

Convert constructor parameter DI → `inject()` field. Requires Angular v14+.

## Eligibility

A constructor parameter is eligible iff it has an access modifier: `private` / `protected` / `public` / `readonly` (these are the parameters TS turns into class fields). Plain params (no modifier) stay in the constructor.

## Conversion

For each eligible `<modifiers> <name>: <Type>`:
- Create field: `<modifiers> <name> = inject(<Type>);`
- Preserve modifier(s) verbatim (`private readonly` stays `private readonly`).
- Remove the param from the constructor signature.

Same name already exists as a field → leave + TODO `// TODO(ng-migrate): to-inject: name collision for <name> — manual review`.

## Decorator translations

| Decorator                  | Conversion                                          |
| -------------------------- | --------------------------------------------------- |
| `@Inject(TOKEN) x`         | `x = inject(TOKEN)` (token type inferred)           |
| `@Optional() x: T`         | `x = inject(T, { optional: true })`                 |
| `@Self() x: T`             | `x = inject(T, { self: true })`                     |
| `@SkipSelf() x: T`         | `x = inject(T, { skipSelf: true })`                 |
| `@Host() x: T`             | `x = inject(T, { host: true })`                     |

Multiple decorators → merge into one options object. Unknown DI decorator → leave + TODO.

## Constructor body

After moving params out:
- Body has any statement (incl. `super()`) → keep `constructor()` with empty params.
- Body empty + class doesn't extend → remove the constructor.
- Body empty + class extends → keep `constructor() { super(); }`.
- `super(...)` uses a constructor param's value → leave that param in the signature, don't convert it; add TODO `// TODO(ng-migrate): to-inject: parameter <name> forwarded to super(...) — manual review`.

## Field placement (strict)

Inject block sits **at the constructor's position** (immediately above it, or where it used to be if removed). Order of class members after migration:

1. Plain fields, signals (`signal()`, `input()`, `input.required()`, `model()`, `output()`, `viewChild()`, `contentChild()`, `toSignal()`)
2. `computed(...)`
3. Getters / setters
4. **Inject block** (pre-existing + new injects, original parameter order, contiguous; `take-until-destroyed` later appends `destroyRef` here)
5. Constructor (if kept)
6. Methods

Do not interleave the inject block with anything else.

## Imports

Add `inject` from `@angular/core` if missing. Keep imports of injected types — they remain as `inject(<Type>)` arguments.

## Don't touch

Visibility, names, types, call sites (`this.foo.x()` continues to work). `Injector.get(...)` patterns are out of scope.
