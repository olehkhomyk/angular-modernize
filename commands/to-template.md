---
description: Migrate template control flow — *ngIf → @if, *ngFor → @for, [ngSwitch] → @switch.
argument-hint: <file-or-folder>
---

Run the `template-control-flow` operation from the `angular-modernize` skill on: $ARGUMENTS

If $ARGUMENTS is empty, ask the user which file or folder. Do not run on the whole project without a target.

Read **before editing** and follow precisely:
- `skills/angular-modernize/references/shared-conventions.md`
- `skills/angular-modernize/references/template-control-flow.md`

Apply to every `.html` file in the target (and inline templates inside `*.ts` `@Component({ template: '...' })`). Do NOT introduce `@defer` blocks unless the user explicitly marks a region. Do NOT migrate third-party structural directives.

Report changed files and any `<!-- TODO(ng-migrate): control-flow: ... -->` comments. Ask before running verification.
