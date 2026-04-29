# Shared conventions

Apply across all five operations.

## TODO comment format

When a transformation cannot be made safely, leave the original code in place and add a comment so the user can grep for it later.

- TypeScript / JS: `// TODO(ng-migrate): <operation>: <reason>`
- HTML template: `<!-- TODO(ng-migrate): <operation>: <reason> -->`

`<operation>` is one of: `take-until-destroyed`, `subscribe-object-form`, `signals-ts`, `signals-template`, `control-flow`.

`<reason>` is a short, concrete sentence — what is ambiguous and what a human should decide. Examples:
- `// TODO(ng-migrate): signals-ts: in-place mutation of nested object — confirm intended write semantics`
- `<!-- TODO(ng-migrate): control-flow: structural directive from third-party lib (cdkVirtualFor) — left as-is -->`

After the run, list every file where TODOs were inserted in the summary.

## Injection detection

Before adding any `private foo = inject(Foo);` line, check if it already exists.

- Search the class for `inject(<Type>)` — if found, do nothing.
- Otherwise add the injection at the placement point defined by the operation.
- Ensure `inject` is imported from `@angular/core`. Add the import if missing.

## Paired-file resolution

For `signals` (and only `signals`), when migrating a TS component file, locate the paired template:

1. Same basename, `.html` extension, in the same directory.
   `foo.component.ts` → `foo.component.html`
2. If `templateUrl: '...'` is set in `@Component(...)`, resolve relative to the TS file.
3. If `template: \`...\`` is inline, migrate the inline template string in place.
4. If neither is found, skip the template step and note it in the summary: "no paired template found for `<file>`."

## Imports

After every operation:
- Remove imports from `@angular/core` (or other Angular packages) that are no longer referenced.
- Add any imports the migration introduces (`signal`, `inject`, `DestroyRef`, `takeUntilDestroyed`).
  - `takeUntilDestroyed` is imported from `@angular/core/rxjs-interop`.
  - `signal`, `inject`, `DestroyRef` are imported from `@angular/core`.

## Angular version detection

Read `package.json` (walk up from the target if needed) and look at `dependencies['@angular/core']`. Strip a leading `^` or `~`, take the major version.

- `< 17`: warn the user that template control flow (`@if` / `@for`) requires v17+. Ask before proceeding.
- `< 16`: warn that `takeUntilDestroyed` and `DestroyRef` require v16+.
- If `package.json` is unreachable, assume v17+ and note the assumption in the run summary.

## File targeting

- Single file → migrate that file only.
- List of files → migrate each.
- Folder → walk recursively. Apply each operation to applicable file types:
  - `take-until-destroyed`, `subscribe-object-form`, `signals-ts` → `*.ts` (excluding `*.spec.ts` unless the user explicitly opts in)
  - `signals-template`, `control-flow` → `*.html` (and inline templates inside `*.ts`)
- Never expand to the whole project / cwd without an explicit target from the user.

## Output style

- Edit files directly with `Edit` or `Write`. Do not print migrated code into the chat.
- Final message: a short list of changed files, count of TODOs inserted (with locations), and the verification-command prompt.
