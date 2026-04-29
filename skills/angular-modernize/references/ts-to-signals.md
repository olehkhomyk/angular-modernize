# Operation: signals-ts (component class → signals)

Convert eligible class fields to Angular signals and update **all** read/write sites in the file. Behavior must remain identical.

After running this on a `.ts` file, immediately run `signals-template` on the paired template (see `shared-conventions.md` § Paired-file resolution).

## 1. Property declaration → signal

**Be aggressive: every plain stateful class field becomes a signal**, including fields without an initializer. Only fields in the skip list (1d) are left alone. After the pass, the only non-signal plain fields remaining must match the skip list.

### 1a. Fields **with** an initializer

```ts
property: string = 'someValue';   // → property = signal<string>('someValue');
property = 'someValue';           // → property = signal<string>('someValue');
count: number = 0;                // → count = signal<number>(0);
items: Item[] = [];               // → items = signal<Item[]>([]);
config = { enabled: true };       // → config = signal<{ enabled: boolean }>({ enabled: true });
```

- Preserve the explicit type as the generic.
- If no explicit type, infer from the initializer. Avoid `any` unless unavoidable.

### 1b. Fields **without** an initializer (typed)

These are still state — convert them. Use `signal<T | undefined>(undefined)` to preserve the original "uninitialized" semantics. **Do not** invent default values like `false`, `0`, or `''` — that changes runtime behavior.

```ts
data: any;                        // → data = signal<any>(undefined);
form: UntypedFormGroup;           // → form = signal<UntypedFormGroup | undefined>(undefined);
display: boolean;                 // → display = signal<boolean | undefined>(undefined);
isAddDialog: boolean;             // → isAddDialog = signal<boolean | undefined>(undefined);
dialogAdminId: string;            // → dialogAdminId = signal<string | undefined>(undefined);
selectedRow: Row;                 // → selectedRow = signal<Row | undefined>(undefined);
```

- For `any`-typed fields, the union is redundant — use `signal<any>(undefined)`.
- Definite-assignment fields (`field!: T`) follow the same rule: drop the `!`, use `signal<T | undefined>(undefined)`.

### 1c. Untyped fields without an initializer (rare)

```ts
mystery;                          // ← cannot infer type
```

Leave as-is and add:
`// TODO(ng-migrate): signals-ts: cannot infer type for uninitialized untyped field — manual review`

### 1d. Skip list — do **NOT** convert these

- Fields decorated with `@Input()`, `@Output()`, `@ViewChild()`, `@ViewChildren()`, `@ContentChild()`, `@ContentChildren()`, `@HostBinding()`.
- Fields already in signal-form: `signal(...)`, `computed(...)`, `input(...)`, `input.required(...)`, `model(...)`, `output(...)`, `viewChild(...)`, `viewChildren(...)`, `contentChild(...)`, `toSignal(...)`.
- Fields whose initializer is `inject(...)`.
- `readonly` fields and `static` fields.
- Subjects / observables: `Subject`, `BehaviorSubject`, `ReplaySubject`, `AsyncSubject`, `Observable` (and subclasses). They are reactive primitives already.
- Method-like fields: getters, setters, methods, arrow-function method fields (`fn = () => {...}`, `fn: (x: T) => U = ...`).
- **Function-reference aliases** (a frequent miss): a field whose declared type is a function signature, OR whose initializer is a bare identifier that resolves to a function. Example:
  ```ts
  insertSpacesInCamelCaseWords: (item: string) => any = insertSpacesInCamelCaseWords;
  ```
  This is a method alias, not state. Skip.
- Constructor-injected fields written as `constructor(private foo: Foo) {}` — those are not regular class fields and are handled by the separate `to-inject` operation.

## 2. Naming

Keep the original property name. Do **not** add `Signal` suffixes. If a name collides (rare), pick a consistent suffix and update **every** reference in the file accordingly.

## 3. Reads → call

Every read site must become a function call:

```ts
this.property                // → this.property()
fn(this.property)            // → fn(this.property())
this.count + 1               // → this.count() + 1
`${this.title} updated!`     // → `${this.title()} updated!`
if (this.items.length)       // → if (this.items().length)
```

Including:
- Template literals
- Conditions (`if`, ternaries)
- Function arguments
- Property/array access (`this.items[0]` → `this.items()[0]`)
- Destructuring is unsafe — convert `const { property } = this;` to `const property = this.property();` (snapshot) and add a TODO if the original code likely depended on later updates.

## 4. Writes → `.set(...)`

Plain assignments become `.set(...)`:

```ts
this.property = value;       // → this.property.set(value);
```

## 5. Compound assignments → `.update(...)`

Anything that depends on the previous value uses `.update(...)`:

```ts
this.count += 1;                       // → this.count.update(v => v + 1);
this.text += suffix;                   // → this.text.update(v => v + suffix);
this.count++;                          // → this.count.update(v => v + 1);
++this.count;                          // → this.count.update(v => v + 1);
this.count--;                          // → this.count.update(v => v - 1);
this.title = `${this.title} updated!`; // → this.title.update(v => `${v} updated!`);
this.list = [...this.list, x];         // → this.list.update(v => [...v, x]);
```

## 6. In-place mutations → immutable `.update(...)`

```ts
this.items.push(x);
// → this.items.update(arr => [...arr, x]);

this.items.splice(i, 1);
// → this.items.update(arr => arr.filter((_, idx) => idx !== i));

this.items[i] = x;
// → this.items.update(arr => arr.map((it, idx) => idx === i ? x : it));

this.config.enabled = true;
// → this.config.update(cfg => ({ ...cfg, enabled: true }));
```

If a mutation is too complex to confidently convert (e.g. deep nested mutation, mutation passed to a library), leave the original line and add:
`// TODO(ng-migrate): signals-ts: in-place mutation — manual review`

## 7. Imports

Ensure `signal` is imported from `@angular/core`. Add if missing. Remove unused imports introduced by deleted plain fields, if any.

```ts
import { signal } from '@angular/core';
```

If `computed` / `effect` are needed by transformations, import those too.

## 8. Don't break framework features

Do **not** convert these — they are already reactive primitives or framework hooks:
- Fields assigned via `input()`, `model()`, `output()`, `viewChild()`, `contentChild()`, `signal()`, `computed()`, `toSignal()`.
- `@Input()` / `@Output()` decorated fields (legacy decorator-based I/O).
- Fields whose initializer comes from `inject(...)`.
- Static or `readonly` fields.

## Validation checklist

- File compiles (no missing imports, no stale references).
- Every former direct read site is now a call.
- Every former direct write is `.set` or `.update`.
- No mutations on the inner value of a signal remain.
- Paired template was queued for `signals-template`.

## Coverage check (mandatory before finishing)

After editing, scan the class body for any remaining plain field declarations of the form:
- `<name>: <Type>;`
- `<name>: <Type> = <value>;`
- `<name> = <value>;`
- `<name>!: <Type>;`

For each one found, it MUST match one of the skip-list categories in 1d. If you find a plain stateful field that doesn't match the skip list, you missed it — go back and convert it. **Do not finish the operation until every non-skip-list field is a signal.**

Common fields that get accidentally missed (always convert these unless decorated/injected):
- form group / form control fields (`form: FormGroup;`, `form: UntypedFormGroup;`)
- dialog / display flags (`display: boolean;`, `visible: boolean;`, `isOpen: boolean;`)
- selected-state fields (`selectedX: T;`, `currentY: T;`)
- IDs / strings (`dialogId: string;`, `userId: number;`)
- generic data containers (`data: any;`, `rows: T[];`)
