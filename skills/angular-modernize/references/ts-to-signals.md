# Operation: signals-ts (component class → signals)

Convert eligible class fields to Angular signals and update **all** read/write sites in the file. Behavior must remain identical.

After running this on a `.ts` file, immediately run `signals-template` on the paired template (see `shared-conventions.md` § Paired-file resolution).

## 1. Property declaration → signal

For each plain class field with an initializer, convert it to a `signal<T>(initial)`:

```ts
property: string = 'someValue';   // → property = signal<string>('someValue');
property = 'someValue';           // → property = signal<string>('someValue');
count: number = 0;                // → count = signal<number>(0);
items: Item[] = [];               // → items = signal<Item[]>([]);
config = { enabled: true };       // → config = signal<{ enabled: boolean }>({ enabled: true });
```

- Preserve the explicit type as the signal's generic argument when present.
- If no explicit type, infer from the initializer. Avoid `any` unless unavoidable.
- Do **not** convert `readonly` fields, `static` fields, fields without initializers, fields decorated with `@Input()`, `@Output()`, `@ViewChild()`, `@ContentChild()`, fields already initialized via `inject(...)`, or fields that are already `signal(...)` / `computed(...)` / `input()` / `model()` / `output()` / `viewChild()`.
- Do **not** convert getters/setters, methods, or arrow-function fields used as methods (`fn = () => {...}`).

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
