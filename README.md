# angular-modernize

A Claude Code **plugin** that performs six mechanical, behavior-preserving Angular migrations from legacy patterns to modern (v17–v18+) idioms.

## Operations

| Operation                | What it does                                                                                          | File types          |
| ------------------------ | ----------------------------------------------------------------------------------------------------- | ------------------- |
| `subscribe-object-form`  | Deprecated `subscribe(next, error, complete)` → `subscribe({ next, error, complete })`.               | `*.ts`              |
| `to-inject`              | Constructor parameter DI (`private foo: Foo`) → `private foo = inject(Foo)`. Block placed at the constructor's position, after signals/computed/getters. | `*.ts` |
| `take-until-destroyed`   | Adds `takeUntilDestroyed(this.destroyRef)` to every `.subscribe()`. Injects `DestroyRef` if needed.   | `*.ts`              |
| `signals` (TS + HTML)    | Class fields → `signal()` (covers uninitialized fields). `@Input()` → `input()` / `input.required()`. `@Output()` → `output()`. Paired template gets `()` reads and two-way bindings split into `[prop]` + `(propChange)`. FormGroup/FormControl `@Input`s left as TODO. | `*.ts` + paired `*.html` |
| `template-control-flow`  | `*ngIf`/`*ngFor`/`[ngSwitch]` → `@if`/`@for`/`@switch`.                                              | `*.html`            |
| **`/modernize`** (combo) | Runs all five above in canonical order: subscribe → inject → cleanup → signals → control-flow.        | both                |

## Install — one-line plugin install

Inside Claude Code:

```
/plugin marketplace add olehkhomyk/angular-modernize
/plugin install angular-modernize@angular-modernize
```

To update later:
```
/plugin marketplace update angular-modernize
```

To uninstall:
```
/plugin uninstall angular-modernize
```

> Requires Claude Code recent enough to support marketplace installs (Node 20+). If you hit *"This plugin uses a source type your Claude Code version does not support"*, update Claude Code: `npm i -g @anthropic-ai/claude-code@latest`.

## Install — manual fallback (no plugin manager)

Copy or symlink the inner skill folder into your skills directory:

```bash
git clone https://github.com/olehkhomyk/angular-modernize.git ~/code/angular-modernize
ln -s ~/code/angular-modernize/skills/angular-modernize ~/.claude/skills/angular-modernize
```

This path doesn't get the `/modernize` slash command — only the skill itself. For the command, install via the plugin manager.

## Usage

There is **one slash command** for the all-in combo:

```text
/modernize src/app/users/user.component.ts
/modernize src/app/users/
```

Everything else is triggered by **natural language** — name a target file or folder and describe what you want, the skill picks the right operation(s):

```text
"Migrate src/app/user/user.component.ts to signals"
"Convert constructor injection to inject() in src/app/api.service.ts"
"Add takeUntilDestroyed to src/app/user/"
"Convert subscribe callbacks to object form in src/app/services/api.service.ts"
"Convert *ngIf to @if in src/app/dashboard/"
"Modernize src/app/users/"   ← same as /modernize
```

The skill always requires a target. It refuses to run on the whole project / cwd without one.

## How `to-signals` handles tricky cases

- **Uninitialized typed fields** become `signal<T | undefined>(undefined)` to preserve original "uninitialized" semantics. No invented defaults like `false`/`0`/`''`.
- **`@Input()`** → `input<T>()` / `input.required<T>()` / `input<T>(default, { alias, transform })`.
- **`@Output()`** → `output<T>()` / `output<T>({ alias })`. Call sites `.emit(x)` keep working.
- **`@Input()` of `FormGroup` / `FormControl` / `FormArray` / `AbstractControl`** is left untouched with a `TODO(ng-migrate)` — these are reference handoffs and parents often share the instance.
- **Two-way bindings** (`[(visible)]="display"`) get split in the template: `[visible]="display()" (visibleChange)="display.set($event)"`. The TS field is converted to a signal as usual.
- **Setter-style `@Input set foo(v) { ... }`** is left as-is with a TODO (manual review — usually means converting to `input()` + `effect()` or `computed()`).
- **Function-reference aliases** (`fn: (x: T) => U = otherFn`) are detected as method aliases, not state, and skipped.
- **Subjects / Observables** are skipped (already reactive primitives).

## How `to-inject` places the inject block

After migration, the class layout is:

```ts
export class Foo extends Bar {
  // signals / inputs / outputs
  loading = signal(true);
  count = input(0);
  saved = output<User>();

  // computed
  total = computed(() => this.count() * 2);

  // getters / setters
  get isReady() { return !this.loading(); }

  // ── inject block lands here (where the constructor was) ──
  private fooSvc = inject(FooService);
  private barSvc = inject(BarService);
  private destroyRef = inject(DestroyRef);

  // constructor — kept only if its body still has logic
  constructor() {
    super();
    this.initForm();
  }

  // methods
  ngOnInit() { ... }
}
```

Constructor parameters with access modifiers (`private`/`protected`/`public`/`readonly`) become inject fields. Decorators are translated to options:
- `@Optional()` → `{ optional: true }`
- `@SkipSelf()` → `{ skipSelf: true }`
- `@Self()` → `{ self: true }`
- `@Host()` → `{ host: true }`
- `@Inject(TOKEN)` → `inject(TOKEN)` directly

If the constructor body becomes empty AND no `super()` is needed, the constructor is removed.

## What the skill will NOT do

- Run on the whole project / cwd without you naming a target.
- Refactor logic, rename variables, or reformat code.
- Touch third-party structural directives (`*cdkVirtualFor`, `*matRowDef`, `*ngrxLet`, …).
- Add `@defer` blocks unless you explicitly mark a region.
- Run a build or typecheck without asking you first.
- Convert constructor parameters that lack an access modifier (those are local-only).
- Convert subscribes on non-RxJS sources (event emitters, custom subscribe).
- Convert `@ViewChild` / `@ContentChild` / `@HostBinding` (out of scope for this version).

## Verification

After each operation, the skill asks whether to run a verification command (it auto-detects `typecheck`/`build`/`lint` from `package.json` scripts; otherwise it asks). You can always say no.

## Unsafe / ambiguous transformations

When a transformation can't be made safely, the skill leaves the original code in place and inserts a comment:

```ts
// TODO(ng-migrate): <operation>: <reason>
```
```html
<!-- TODO(ng-migrate): <operation>: <reason> -->
```

Grep for `TODO(ng-migrate)` after a run to find them.

## Angular version requirements

- `to-inject` — v14+ (recommended v16+)
- `take-until-destroyed` — v16+
- `signals` — v16+ for `signal()`/`computed()`; v17.1+ for `input()`/`output()`
- `template-control-flow` — v17+
- `subscribe-object-form` — any modern RxJS version

The skill reads `package.json` for `@angular/core` and warns before running anything that requires a newer version.

## Repository layout

```
angular-modernize/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── README.md
├── commands/
│   └── modernize.md                            # the only slash command
└── skills/
    └── angular-modernize/
        ├── SKILL.md
        └── references/
            ├── shared-conventions.md
            ├── subscribe-object-form.md
            ├── to-inject.md
            ├── take-until-destroyed.md
            ├── ts-to-signals.md
            ├── template-to-signals.md
            └── template-control-flow.md
```

## License

MIT.
