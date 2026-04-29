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

### 1d. `@Input()` → `input<T>()` / `input.required<T>()`

Convert decorator-based inputs to the new signal-input form. **Skip-with-TODO** when the input type is a form reference (see exceptions).

```ts
@Input() name: string;                            // → name = input<string>();
@Input() name: string = 'default';                // → name = input<string>('default');
@Input() count: number = 0;                       // → count = input<number>(0);
@Input({ required: true }) id: string;            // → id = input.required<string>();
@Input({ alias: 'externalName' }) name: string;   // → name = input<string>(undefined, { alias: 'externalName' });
@Input({ transform: numberAttribute }) n: number; // → n = input<number>(undefined, { transform: numberAttribute });
```

- Preserve the property name and the type as the generic.
- Required form (`@Input({ required: true })`) → `input.required<T>()` (no default).
- Options like `alias`, `transform` are forwarded as the second argument.
- Optional inputs without a default keep the type as-is — `input<T>()` returns a `Signal<T | undefined>` automatically; do NOT add `| undefined` to the generic.
- Setter-style `@Input set foo(v: T) { ... }` patterns are NOT auto-converted. Leave the original setter and add:
  `// TODO(ng-migrate): signals-ts: @Input setter — manual review (consider input() + effect() or computed())`

#### Exception: form-reference inputs (skip with TODO)

Do NOT convert `@Input()` whose type extends/is one of:
- `FormGroup`, `UntypedFormGroup`
- `FormControl`, `UntypedFormControl`
- `FormArray`, `UntypedFormArray`
- `AbstractControl`, `UntypedAbstractControl`

Leave the legacy `@Input()` decorator and add:
`// TODO(ng-migrate): signals-ts: @Input is a form reference — left as decorator input (manual review for signal conversion)`

Reason: passing a form instance is a reference handoff and the parent often binds to the same instance for two-way state; converting blindly to `input()` can change identity semantics and break parent bindings.

### 1e. `@Output()` → `output<T>()`

Convert decorator-based outputs to the new function form.

```ts
@Output() saved = new EventEmitter<User>();         // → saved = output<User>();
@Output() closed = new EventEmitter<void>();        // → closed = output<void>();
@Output('savedEvent') saved = new EventEmitter<User>();
// → saved = output<User>({ alias: 'savedEvent' });
```

- The new `output()` does not extend `EventEmitter`, but it preserves `.emit(value)` — call sites like `this.saved.emit(user)` keep working unchanged.
- Skip and TODO when the field is used in code paths that rely on `EventEmitter`-specific APIs (`.subscribe(...)`, `.pipe(...)`, casting to `Observable`):
  `// TODO(ng-migrate): signals-ts: @Output is consumed as an Observable — manual review`

### 1f. Skip list — do **NOT** convert these

- Fields decorated with `@ViewChild()`, `@ViewChildren()`, `@ContentChild()`, `@ContentChildren()`, `@HostBinding()`. (These have signal-form equivalents — `viewChild()`, `contentChild()` — but auto-migration is not in scope here.)
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

## 7. Two-way bindings (`[(prop)]="x"`) — do NOT skip

If a field is two-way bound in the paired template (`[(visible)]="display"`, `[(ngModel)]="value"`, etc.), convert the field to a signal anyway. The paired-template pass will split the two-way binding into one-way + event handler. Do not leave a TODO for this case.

The split (handled in `template-to-signals.md`) looks like:
```html
<!-- before -->
[(visible)]="display"
<!-- after -->
[visible]="display()" (visibleChange)="display.set($event)"
```

Special case: `[(ngModel)]` on a form-control that's bound to a primitive field. Same rule — convert the field, split the binding to `[ngModel]="x()" (ngModelChange)="x.set($event)"`. (If the field is a form group/control reference, it's already in the @Input skip list above.)

## 8. Imports

Ensure the right imports from `@angular/core`:
- `signal` — always when fields are converted
- `input`, `output` — when `@Input()` / `@Output()` are converted
- `computed`, `effect` — only if transformations introduce them

Remove unused imports introduced by deleted plain fields. Specifically: if all `@Input()` decorators were converted to `input()`, remove `Input` from the `@angular/core` import. Same for `Output` and `EventEmitter` (the latter only if no remaining usage).

## 9. Don't break framework features

Do **not** apply the plain-field conversion to these (they're handled separately or are already reactive):
- Fields already assigned via `input()`, `model()`, `output()`, `viewChild()`, `contentChild()`, `signal()`, `computed()`, `toSignal()`.
- Fields whose initializer comes from `inject(...)`.
- `static` and `readonly` fields.

`@Input()` / `@Output()` are NOT skipped — they're converted via sections 1d and 1e. The exception is form-reference inputs (FormGroup/FormControl/AbstractControl), which get a TODO instead.

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
