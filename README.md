# angular-modernize

A Claude Code **plugin** that performs five mechanical, behavior-preserving Angular migrations from legacy patterns to modern (v17–v18+) idioms.

## What it does

Five operations, each invokable on its own or together as `modernize`:

| Operation                | What it does                                                                 | File types          |
| ------------------------ | ---------------------------------------------------------------------------- | ------------------- |
| `take-until-destroyed`   | Adds `takeUntilDestroyed(this.destroyRef)` to every `.subscribe()`. Injects `DestroyRef` if needed. | `*.ts`              |
| `subscribe-object-form`  | Converts deprecated `subscribe(next, error, complete)` → `subscribe({ next, error, complete })`. | `*.ts`              |
| `signals`                | Converts class fields to signals and updates every read/write site, plus the paired template.    | `*.ts` + paired `*.html` |
| `template-control-flow`  | `*ngIf`/`*ngFor`/`[ngSwitch]` → `@if`/`@for`/`@switch`.                      | `*.html`            |
| `modernize` (combo)      | Runs all four in order: subscribe-object-form → take-until-destroyed → signals → control-flow. | both                |

## Install (recommended) — one-line plugin install

Inside Claude Code:

```
/plugin marketplace add olehkhomyk/angular-modernize
/plugin install angular-modernize@angular-modernize
```

That's it. The skill is now active in every session.

To update later:
```
/plugin marketplace update angular-modernize
```

To uninstall:
```
/plugin uninstall angular-modernize
```

## Install — alternative (manual clone, no plugin manager)

If you don't want to use the plugin system, clone the **inner** skill folder directly into your skills directory:

```bash
git clone https://github.com/olehkhomyk/angular-modernize.git /tmp/angular-modernize
cp -r /tmp/angular-modernize/skills/angular-modernize ~/.claude/skills/angular-modernize
```

Or as a one-liner symlink (so `git pull` keeps it updated):
```bash
git clone https://github.com/olehkhomyk/angular-modernize.git ~/code/angular-modernize
ln -s ~/code/angular-modernize/skills/angular-modernize ~/.claude/skills/angular-modernize
```

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

## Repository layout

```
angular-modernize/                          # repo root = marketplace root
├── .claude-plugin/
│   ├── plugin.json                         # plugin metadata
│   └── marketplace.json                    # makes this repo its own marketplace
├── README.md
└── skills/
    └── angular-modernize/                  # the actual skill
        ├── SKILL.md                        # router and rules
        └── references/
            ├── shared-conventions.md       # TODO format, paired-file rule, version detection
            ├── take-until-destroyed.md
            ├── subscribe-object-form.md
            ├── ts-to-signals.md
            ├── template-to-signals.md
            └── template-control-flow.md
```

`SKILL.md` is the only file Claude reads up front — it routes to the relevant reference file(s) based on what you asked for.

## License

MIT.
