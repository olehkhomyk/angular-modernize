---
description: Run a comma-separated list of angular-modernize operations on a file or folder. Example — /ng-migrate to-signals,to-template src/app/foo
argument-hint: <ops-comma-separated> <file-or-folder>
---

Parse $ARGUMENTS as: `<comma-separated-ops> <target>`.

The first token is a comma-separated list of operation names. Valid names:
- `to-subscribe`     → subscribe-object-form
- `to-inject`        → constructor DI → inject()
- `to-cleanup`       → takeUntilDestroyed
- `to-signals`       → signals (TS + paired HTML)
- `to-template`      → template control flow
- `modernize`        → all five (use this alone)

The remaining tokens form the target file or folder path.

Examples:
- `/ng-migrate to-signals src/app/foo.component.ts`
- `/ng-migrate to-signals,to-template src/app/foo.component.ts`
- `/ng-migrate to-subscribe,to-cleanup,to-inject src/app/services/`
- `/ng-migrate modernize src/app/users/`

If the ops list or target is missing/ambiguous, ask the user.

Run the listed operations on the target in **this canonical order** (regardless of how the user listed them) — applying each to its applicable file types only:
1. `to-subscribe`
2. `to-inject`
3. `to-cleanup`
4. `to-signals` (also runs paired template signals migration)
5. `to-template`

Read **before editing**: `skills/angular-modernize/references/shared-conventions.md`. Then for each requested op, read its reference file and follow it precisely:
- `skills/angular-modernize/references/subscribe-object-form.md`
- `skills/angular-modernize/references/to-inject.md`
- `skills/angular-modernize/references/take-until-destroyed.md`
- `skills/angular-modernize/references/ts-to-signals.md` + `skills/angular-modernize/references/template-to-signals.md`
- `skills/angular-modernize/references/template-control-flow.md`

After all requested ops finish, give one combined summary: which ops ran, changed files, counts (signals introduced, services injected, subscribes wrapped, control-flow blocks migrated), `TODO(ng-migrate)` lines (file:line), and ask before running verification.
