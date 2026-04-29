# angular-modernize

A Claude Code **plugin** that performs six mechanical, behavior-preserving Angular migrations from legacy patterns to modern (v17–v18+) idioms.

## Operations

| Operation                | What it does                                                                 | File types          |
| ------------------------ | ---------------------------------------------------------------------------- | ------------------- |
| `to-subscribe`           | Converts deprecated `subscribe(next, error, complete)` → `subscribe({ next, error, complete })`. | `*.ts` |
| `to-inject`              | Constructor parameter DI (`private foo: Foo`) → `private foo = inject(Foo)`. | `*.ts`              |
| `to-cleanup`             | Adds `takeUntilDestroyed(this.destroyRef)` to every `.subscribe()`. Injects `DestroyRef` if needed. | `*.ts` |
| `to-signals`             | Class fields → `signal()` (including uninitialized fields), with paired-template `()` updates.   | `*.ts` + paired `*.html` |
| `to-template`            | `*ngIf`/`*ngFor`/`[ngSwitch]` → `@if`/`@for`/`@switch`.                      | `*.html`            |
| `modernize`              | Runs all five in canonical order: subscribe → inject → cleanup → signals → template. | both                |

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

This path doesn't get the slash commands — only the skill itself. For commands, install via the plugin manager.

## Usage — slash commands

Each operation has its own command. **All commands require a target** — file, list of files, or folder. They refuse to run on the whole project without an explicit target.

```text
/to-signals     src/app/user/user.component.ts
/to-subscribe   src/app/api.service.ts
/to-cleanup     src/app/api.service.ts
/to-inject      src/app/user/user.component.ts
/to-template    src/app/dashboard/dashboard.component.html
/modernize      src/app/users/
```

### Comma-separated chaining via `/ng-migrate`

Run multiple operations in one shot:

```text
/ng-migrate to-signals,to-template src/app/foo.component.ts
/ng-migrate to-subscribe,to-cleanup,to-inject src/app/services/
/ng-migrate modernize src/app/users/
```

Operations always run in the safe canonical order regardless of how you list them: `to-subscribe → to-inject → to-cleanup → to-signals → to-template`.

## Usage — natural language

The skill also triggers on natural language without commands:

```text
"Migrate src/app/user/user.component.ts to signals"
"Convert constructor injection to inject() in src/app/api.service.ts"
"Add takeUntilDestroyed to src/app/user/"
"Convert *ngIf to @if in src/app/dashboard/"
"Modernize src/app/users/"
```

When you say "to signals" on a `.ts` file, the skill auto-locates and migrates the paired `.html` template (or inline `template:`) so signal reads use `()`.

## Coverage guarantee for `to-signals`

Every plain stateful class field becomes a signal — including fields without initializers. Uninitialized typed fields become `signal<T | undefined>(undefined)` to preserve "uninitialized" semantics. The skip list is small and explicit:

- decorator-bound fields (`@Input`, `@Output`, `@ViewChild`, etc.)
- already-signal forms (`signal()`, `computed()`, `input()`, `model()`, `viewChild()`, `toSignal()`, …)
- `inject(...)` initializers
- `readonly` / `static`
- `Subject` / `BehaviorSubject` / `Observable` (already reactive)
- methods, getters, setters, function-reference aliases

Each `to-signals` run ends with a mandatory coverage check — if any non-skip-list field is left as a plain field, the skill fixes it before reporting done.

## What the skill will NOT do

- Run on the whole project / cwd without you naming a target.
- Refactor logic, rename variables, or reformat code.
- Touch third-party structural directives (`*cdkVirtualFor`, `*matRowDef`, `*ngrxLet`, …).
- Add `@defer` blocks unless you explicitly mark a region.
- Run a build or typecheck without asking you first.
- Convert constructor parameters that lack an access modifier (those are local-only).
- Convert subscribes on non-RxJS sources (event emitters, custom subscribe).

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
- `to-cleanup` — v16+
- `to-signals` — v16+
- `to-template` — v17+
- `to-subscribe` — any modern RxJS version

The skill reads `package.json` for `@angular/core` and warns before running anything that requires a newer version.

## Repository layout

```
angular-modernize/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── README.md
├── commands/                                   # slash commands
│   ├── to-signals.md
│   ├── to-subscribe.md
│   ├── to-cleanup.md
│   ├── to-inject.md
│   ├── to-template.md
│   ├── modernize.md
│   └── ng-migrate.md
└── skills/
    └── angular-modernize/
        ├── SKILL.md
        └── references/
            ├── shared-conventions.md
            ├── take-until-destroyed.md
            ├── subscribe-object-form.md
            ├── to-inject.md
            ├── ts-to-signals.md
            ├── template-to-signals.md
            └── template-control-flow.md
```

## License

MIT.
