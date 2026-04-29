# signals-template

Update template reads to call signals (`prop` → `prop()`) and split two-way bindings.

## Reads → calls

For each signal-backed identifier `prop` (i.e. fields that became signals in the paired TS pass):

```html
{{ prop }}                  → {{ prop() }}
{{ prop + 'x' }}            → {{ prop() + 'x' }}
[value]="prop"              → [value]="prop()"
[disabled]="!prop"          → [disabled]="!prop()"
*ngIf="prop && other"       → *ngIf="prop() && other()"
*ngFor="let x of items"     → *ngFor="let x of items()"
{{ items.length }}          → {{ items().length }}
{{ items[0] }}              → {{ items()[0] }}
{{ user.name }}             → {{ user().name }}    (only if `user` is a signal)
{{ prop | uppercase }}      → {{ prop() | uppercase }}   (convert input only)
```

Apply to every read in expressions, nested or not. Pipes get the call on their input only, not the pipe itself.

## Two-way bindings → split

For every signal-backed `[(name)]="x"`:
```
[(name)]="x"   →   [name]="x()" (nameChange)="x.set($event)"
```
Examples: `[(visible)]`, `[(checked)]`, `[(value)]`, `[(ngModel)]`. Keep the new `(nameChange)` right after the new `[name]`.

If `x` is a non-signal field (skipped in TS), leave + `<!-- TODO(ng-migrate): signals-template: two-way binding on non-signal field '<x>' — manual review -->`.

## Skip

- Method calls (`(click)="doThing()"`) — methods already use `()`.
- Event handler bodies — only convert reads of signals inside argument expressions: `(click)="save(prop)"` → `(click)="save(prop())"` if `prop` is a signal.
- Template-local vars: `#tplRef`, `let-x`, `let i = $index`, `as v`, loop locals.
- Anything not a signal.

## Standalone mode (no preceding TS pass)

Convert only identifiers you can confidently identify as class-field signals. Otherwise leave + `<!-- TODO(ng-migrate): signals-template: cannot determine if '<x>' is a signal — manual review -->`.

## Formatting

Keep whitespace, line breaks, attribute order. Only changed tokens differ from the original.
