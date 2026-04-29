# signals-ts

Convert eligible class fields to signals and update every read/write site. Behavior must remain identical. Run `signals-template` on the paired template afterwards.

## 1. Field → signal

Be aggressive: every plain stateful field becomes a signal unless it matches the skip list (§ 1f).

### 1a. With initializer
```ts
property: T = value;        // → property = signal<T>(value);
property = value;           // → property = signal<inferred>(value);
items: Item[] = [];         // → items = signal<Item[]>([]);
```
Preserve explicit type as the generic. Infer when missing; avoid `any` unless unavoidable.

### 1b. Without initializer (typed) — convert anyway
Use `signal<T | undefined>(undefined)`. Do NOT invent defaults like `false` / `0` / `''` — that changes runtime behavior.
```ts
data: any;                  // → data = signal<any>(undefined);
form: UntypedFormGroup;     // → form = signal<UntypedFormGroup | undefined>(undefined);
display: boolean;           // → display = signal<boolean | undefined>(undefined);
```
- `any` skips the union: `signal<any>(undefined)`.
- Definite assignment `field!: T` → drop `!`, use `signal<T | undefined>(undefined)`.

### 1c. Untyped + uninitialized — leave + TODO
`mystery;` → leave + `// TODO(ng-migrate): signals-ts: cannot infer type — manual review`.

### 1d. `@Input()` → `input<T>()`
```ts
@Input() name: T;                            // → name = input<T>();
@Input() name: T = 'd';                      // → name = input<T>('d');
@Input({ required: true }) id: string;       // → id = input.required<string>();
@Input({ alias: 'x' }) name: T;              // → name = input<T>(undefined, { alias: 'x' });
@Input({ transform: numberAttribute }) n: number; // → n = input<number>(undefined, { transform: numberAttribute });
```
Optional inputs: `input<T>()` already returns `Signal<T | undefined>` — do NOT add `| undefined` to the generic.

`@Input set foo(v: T) { ... }` → leave + TODO `// TODO(ng-migrate): signals-ts: @Input setter — manual review (consider input() + effect()/computed())`.

**Skip-with-TODO** when `@Input` type is `FormGroup` / `UntypedFormGroup` / `FormControl` / `UntypedFormControl` / `FormArray` / `UntypedFormArray` / `AbstractControl` / `UntypedAbstractControl`. Leave the decorator and add `// TODO(ng-migrate): signals-ts: @Input is a form reference — left as decorator input (manual review)`. Reason: parents share the form instance and converting can break bindings.

### 1e. `@Output()` → `output<T>()`
```ts
@Output() saved = new EventEmitter<User>();   // → saved = output<User>();
@Output() closed = new EventEmitter<void>();  // → closed = output<void>();
@Output('aliasName') saved = new EventEmitter<User>();
// → saved = output<User>({ alias: 'aliasName' });
```
Call sites `.emit(x)` keep working. Skip-with-TODO if the output is consumed as Observable (`.subscribe(...)` / `.pipe(...)` on `this.event`).

### 1f. Skip list (do NOT convert)

- Decorators we don't migrate here: `@ViewChild`, `@ViewChildren`, `@ContentChild`, `@ContentChildren`, `@HostBinding`.
- Already-signal forms: `signal()`, `computed()`, `input()`, `input.required()`, `model()`, `output()`, `viewChild()`, `viewChildren()`, `contentChild()`, `toSignal()`.
- `inject(...)` initializers.
- `readonly` and `static`.
- `Subject` / `BehaviorSubject` / `ReplaySubject` / `AsyncSubject` / `Observable` (already reactive).
- Method-like fields: getters, setters, methods, arrow-function method fields (`fn = () => {...}`, `fn: (x: T) => U = ...`).
- Function-reference aliases: type is a function signature, OR initializer is a bare identifier resolving to a function. Example: `myHelper: (x: T) => U = myHelper;` — method alias, not state.
- Constructor-injected params (`constructor(private foo: Foo)`) — handled by `to-inject`.

## 2. Reads → call

Every direct read of a converted field becomes a function call:
```ts
this.x                  // → this.x()
this.x + 1              // → this.x() + 1
fn(this.x)              // → fn(this.x())
`${this.x}`             // → `${this.x()}`
this.items.length       // → this.items().length
this.items[0]           // → this.items()[0]
this.user.name          // (if `user` is a signal) → this.user().name
```
Destructuring is a snapshot — `const { x } = this;` → `const x = this.x();` and add a TODO if the original code likely needed live updates.

## 3. Writes → `.set(...)`

```ts
this.x = v;             // → this.x.set(v);
```

## 4. Compound / in-place → `.update(...)`

```ts
this.n += 1;            // → this.n.update(v => v + 1);
this.n++;               // → this.n.update(v => v + 1);
this.n--;               // → this.n.update(v => v - 1);
this.s += 'x';          // → this.s.update(v => v + 'x');
this.t = `${this.t}!`;  // → this.t.update(v => `${v}!`);

this.items.push(x);     // → this.items.update(arr => [...arr, x]);
this.items[i] = x;      // → this.items.update(arr => arr.map((it, idx) => idx === i ? x : it));
this.cfg.flag = true;   // → this.cfg.update(c => ({ ...c, flag: true }));
```
Complex/nested mutation you can't safely rewrite → leave + TODO `// TODO(ng-migrate): signals-ts: in-place mutation — manual review`.

## 5. Two-way bindings — convert anyway

Don't skip a field just because it's two-way bound in the template (`[(visible)]="display"`, `[(ngModel)]="text"`). Convert the field; the template pass splits the binding into `[name]="x()"` + `(nameChange)="x.set($event)"`.

## 6. Imports

Add as needed: `signal`, `computed`, `effect`, `input`, `output` from `@angular/core`. Remove `Input` / `Output` if all decorators were converted, and `EventEmitter` if it has no other usage.

## Coverage check (mandatory)

After editing, scan the class body. Any plain field declaration of the form `<name>: <T>;`, `<name>: <T> = v;`, `<name> = v;`, `<name>!: <T>;` MUST match the skip list (§ 1f). If a non-skip-list field is left plain, that's a bug — go back and convert it.

Common misses: form-related fields (`form: FormGroup;`), display flags (`display: boolean;`, `visible: boolean;`), selected state (`selectedX: T;`), IDs / strings, generic data containers.
