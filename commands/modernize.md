---
description: Run all five Angular migrations on a file or folder, in safe order.
argument-hint: <file-or-folder>
---

Run **all** angular-modernize operations on: $ARGUMENTS

If $ARGUMENTS is empty, ask the user which file or folder. Do not run on the whole project without a target.

Read **before editing**:
- `skills/angular-modernize/references/shared-conventions.md`

Then run, in this order, on the same target — apply each operation to its applicable file types only:

1. **subscribe-object-form** (`*.ts`) — pure syntactic, safest first.
   - Reference: `skills/angular-modernize/references/subscribe-object-form.md`
2. **to-inject** (`*.ts`) — constructor DI → `inject()`.
   - Reference: `skills/angular-modernize/references/to-inject.md`
3. **take-until-destroyed** (`*.ts`) — adds `DestroyRef` cleanup.
   - Reference: `skills/angular-modernize/references/take-until-destroyed.md`
4. **signals** (`*.ts` + paired `*.html`) — biggest change; run coverage check at end.
   - References: `skills/angular-modernize/references/ts-to-signals.md` + `skills/angular-modernize/references/template-to-signals.md`
5. **template-control-flow** (`*.html`) — `*ngIf`/`*ngFor`/`[ngSwitch]` → new syntax.
   - Reference: `skills/angular-modernize/references/template-control-flow.md`

After all five passes, report a single combined summary: changed files, signals introduced, services injected, subscribes wrapped, control-flow blocks migrated, and any `TODO(ng-migrate)` lines (with file:line). Ask the user before running verification.
