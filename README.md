# angular-modernize

A Claude Code skill that performs five mechanical, behavior-preserving Angular migrations from legacy patterns to modern (v17–v18+) idioms.

## What it does

Five operations, each invokable on its own or together as `modernize`:

| Operation                | What it does                                                                 | File types          |
| ------------------------ | ---------------------------------------------------------------------------- | ------------------- |
| `take-until-destroyed`   | Adds `takeUntilDestroyed(this.destroyRef)` to every `.subscribe()`. Injects `DestroyRef` if needed. | `*.ts`              |
| `subscribe-object-form`  | Converts deprecated `subscribe(next, error, complete)` → `subscribe({ next, error, complete })`. | `*.ts`              |
| `signals`                | Converts class fields to signals and updates every read/write site, plus the paired template.    | `*.ts` + paired `*.html` |
| `template-control-flow`  | `*ngIf`/`*ngFor`/`[ngSwitch]` → `@if`/`@for`/`@switch`.                      | `*.html`            |
| `modernize` (combo)      | Runs all four in order: subscribe-object-form → take-until-destroyed → signals → control-flow. | both                |

## Install

Clone (or copy) the `angular-modernize/` folder into your skills directory:

```bash
# personal install
cp -r angular-modernize ~/.claude/skills/

# or as part of a project's local skills
cp -r angular-modernize <project>/.claude/skills/
```

Restart Claude Code or open a new session — the skill is auto-discovered from `SKILL.md`.

## Usage

The skill triggers on natural language. **You must always name a target** — a file, a list of files, or a folder. The skill will refuse to run on the whole project without an explicit target.

```text
# single operation, single file
"Migrate src/app/user/user.component.ts to signals"
"Add takeUntilDestroyed to src/app/api.service.ts"
"Convert *ngIf to @if in src/app/dashboard/dashboard.component.html"
"Migrate subscribe calls in src/app/api.service.ts to the object form"

# whole folder
"Modernize everything under src/app/users/"

# combo on one file
"Modernize src/app/user/user.component.ts"
```

When you say "to signals" on a `.ts` file, the skill will also migrate the paired `.html` template (same basename, same folder, or whatever `templateUrl` resolves to) so signal reads use `()` in the template.

## What it will NOT do

- Run on the whole project / cwd without you naming a target.
- Refactor logic, rename variables, or reformat code.
- Touch third-party structural directives (`*cdkVirtualFor`, `*matRowDef`, `*ngrxLet`, …).
- Add `@defer` blocks unless you explicitly mark a region.
- Run a build or typecheck without asking you first.

## Verification

After each operation finishes, the skill will ask whether to run a verification command (it tries to detect `typecheck`/`build`/`lint` from your `package.json` scripts; otherwise it asks). You can always say no.

## Unsafe / ambiguous transformations

If the skill cannot make a transformation safely (e.g. a deeply nested in-place mutation, a `subscribe` on something that may or may not be RxJS, an `*ngIf` with an unusual third-party directive), it leaves the original code in place and inserts a comment in this format:

```ts
// TODO(ng-migrate): <operation>: <reason>
```
```html
<!-- TODO(ng-migrate): <operation>: <reason> -->
```

Grep for `TODO(ng-migrate)` after a run to find them.

## Angular version

The skill checks `package.json` for `@angular/core`. If it's older than the version a given operation requires (e.g. `@if`/`@for` need v17+, `takeUntilDestroyed` needs v16+), it warns before running.

## Layout

```
angular-modernize/
├── SKILL.md                              # router and rules
├── README.md                             # this file
└── references/
    ├── shared-conventions.md             # TODO format, paired-file rule, version detection
    ├── take-until-destroyed.md
    ├── subscribe-object-form.md
    ├── ts-to-signals.md
    ├── template-to-signals.md
    └── template-control-flow.md
```

`SKILL.md` is the only file Claude reads up front — it routes to the relevant reference file(s) based on what you asked for.

## License

MIT.
