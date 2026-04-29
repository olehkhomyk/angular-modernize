---
description: Run all Angular migrations on a file or folder. One pass per file.
argument-hint: <file-or-folder>
---

Target: $ARGUMENTS

If $ARGUMENTS is empty, ask which file or folder. Never run on the whole project without a target.

Read the skill router and references **once** at the start of the run:
- `skills/angular-modernize/SKILL.md`
- `skills/angular-modernize/references/shared-conventions.md`
- `skills/angular-modernize/references/subscribe-object-form.md`
- `skills/angular-modernize/references/to-inject.md`
- `skills/angular-modernize/references/take-until-destroyed.md`
- `skills/angular-modernize/references/ts-to-signals.md`
- `skills/angular-modernize/references/template-to-signals.md`
- `skills/angular-modernize/references/template-control-flow.md`

Then for each target file:
1. `Read` the file ONCE.
2. Apply, in memory, in canonical order — only the ops applicable to that file type:
   1. `subscribe-object-form` (`*.ts`)
   2. `to-inject` (`*.ts`)
   3. `take-until-destroyed` (`*.ts`)
   4. `signals-ts` (`*.ts`) — also locate and migrate the paired template via `signals-template` rules
   5. `signals-template` (`*.html` / inline)
   6. `template-control-flow` (`*.html` / inline)
3. `Write` the file ONCE with the final content. No intermediate `Edit`s. No per-op summaries.

When a `.ts` file triggers `signals-ts`, locate the paired template (paired-file rule), `Read` it, apply `signals-template` and `template-control-flow` in memory, `Write` it once.

After all files are written, output one combined summary:
- Changed files (paths only).
- Counts: signals introduced, services injected, subscribes wrapped, control-flow blocks migrated.
- Every `TODO(ng-migrate)` line as `path:line — <reason>`.

Then ask exactly once: "Run verification? (auto-detected: `<cmd>` from package.json — y/n)". Auto-detect from `package.json` scripts (`typecheck`, `build`, `lint`); if none found, ask for the command.
