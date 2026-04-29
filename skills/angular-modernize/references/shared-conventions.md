# Shared conventions

Cross-cutting rules. Keep brief — apply to every operation.

## TODO format

- TS: `// TODO(ng-migrate): <op>: <reason>`
- HTML: `<!-- TODO(ng-migrate): <op>: <reason> -->`

`<op>` ∈ {`take-until-destroyed`, `subscribe-object-form`, `to-inject`, `signals-ts`, `signals-template`, `control-flow`}. `<reason>` is one short sentence.

## Paired-file rule (signals only)

For each `*.ts` migrated to signals, also migrate its template:
1. Same basename `.html` next to the TS file.
2. If `templateUrl` is set, resolve relative to the TS file.
3. If `template:` inline, migrate the string in place.
4. If none found, note "no paired template" and skip.

## Imports

- Add as needed (from `@angular/core`): `signal`, `inject`, `input`, `output`, `computed`, `effect`, `DestroyRef`. From `@angular/core/rxjs-interop`: `takeUntilDestroyed`.
- Remove `Input` / `Output` / `EventEmitter` if all decorators were converted and no other usages remain.
- Never duplicate.

## Version detection

Read `package.json` `dependencies['@angular/core']`. Strip `^`/`~`, take major. Skip these ops if version is too old:
- `< 14`: `to-inject`
- `< 16`: `take-until-destroyed`, `signals` (signal/computed)
- `< 17.1`: `input()`/`output()` conversion (leave decorators)
- `< 17`: `template-control-flow`

If `package.json` is unreachable, assume v17+ and note the assumption.

## File targeting

- Single file / list → those.
- Folder → recurse; `*.ts` ops on `*.ts`, `*.html` ops on `*.html`. Skip `*.spec.ts` unless opted in.
- Never expand to the whole project without an explicit target.

## Execution model

**One pass per file.** Each file is read once and written once. Apply all in-scope transformations to the in-memory string before writing. Do NOT chain multiple `Edit` tool calls on the same file — that wastes tokens via repeated diagnostics.

- Use `Read` once per file.
- Apply all rules from the relevant reference(s) in memory.
- Use `Write` once at the end with the final content.
- The only exception: when the file is too large to safely include in a single `Write`, fall back to a minimal sequence of `Edit`s.

After all files are written, produce **one** combined summary and **one** verification prompt — not per-operation summaries.
