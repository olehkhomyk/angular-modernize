# Operation: signals-template (HTML reads → call form, two-way bindings → split)

After a TS file has been migrated to signals, its template needs every read of those signals to be a function call (`property` → `property()`). This pass is mostly read-only with one exception: two-way bindings (`[(prop)]="x"`) are split into a one-way bind + event handler so the signal can be `.set(...)` from the template.

## Inputs

The set of identifiers that became signals in the paired TS file. If running standalone (no recent TS pass), assume any identifier referenced in the template that resolves to a class field is a signal candidate, and convert conservatively (see "Conservative mode" below).

## Conversions

For each signal-backed identifier `prop`:

```html
{{ prop }}                            <!-- → {{ prop() }} -->
{{ prop + 'x' }}                      <!-- → {{ prop() + 'x' }} -->
[value]="prop"                        <!-- → [value]="prop()" -->
[disabled]="!prop"                    <!-- → [disabled]="!prop()" -->
[ngClass]="prop"                      <!-- → [ngClass]="prop()" -->
*ngIf="prop"                          <!-- → *ngIf="prop()" -->
*ngIf="prop && other"                 <!-- → *ngIf="prop() && other()" -->
*ngFor="let x of items"               <!-- → *ngFor="let x of items()" -->
{{ obj.field }}                       <!-- → {{ obj().field }}    (only if `obj` itself is a signal) -->
{{ list[0] }}                         <!-- → {{ list()[0] }}      (only if `list` is a signal) -->
{{ items.length }}                    <!-- → {{ items().length }} -->
```

In nested expressions, every read of a signal must become a call:

```html
[class.active]="active"               <!-- → [class.active]="active()" -->
{{ user.name + ' ' + user.title }}    <!-- if `user` is a signal: → {{ user().name + ' ' + user().title }} -->
```

## Two-way bindings — split into one-way + event

If a signal-backed property is bound via two-way `[(prop)]="x"`, split it into a one-way read + an event setter. Do this for every two-way binding pointing at a signal:

```html
<!-- before -->
[(visible)]="display"
[(checked)]="isChecked"
[(value)]="amount"
[(ngModel)]="text"

<!-- after -->
[visible]="display()"   (visibleChange)="display.set($event)"
[checked]="isChecked()" (checkedChange)="isChecked.set($event)"
[value]="amount()"      (valueChange)="amount.set($event)"
[ngModel]="text()"      (ngModelChange)="text.set($event)"
```

Rule: `[(name)]="x"` becomes `[name]="x()" (nameChange)="x.set($event)"`. The event name is always `<name>Change` (Angular's two-way binding convention). Preserve attribute order: place the new `(<name>Change)` immediately after the new `[<name>]`.

If `x` is a non-signal field (skipped in the TS pass for some reason), leave the two-way binding intact and add:
`<!-- TODO(ng-migrate): signals-template: two-way binding on non-signal field '<x>' — manual review -->`

## Do NOT

- Add `()` to method calls — methods already use `()`. Only convert plain identifier reads.
- Add `()` to event handlers like `(click)="doThing()"` — those are method calls.
- Convert reads inside event handler **arguments** unless the identifier is a signal: `(click)="save(prop)"` → `(click)="save(prop())"` only when `prop` is a signal.
- Add `.set(...)` / `.update(...)` calls to the template **except** for the two-way binding split rule above.
- Convert template reference variables like `#myInput`, `let-x`, `let i = $index` — those are template-local, not signals.
- Convert local template variables introduced by `*ngFor`, `@for`, `@if (... ; as v)`, etc.
- Convert pipes (`{{ prop | uppercase }}`) — convert only the input: `{{ prop() | uppercase }}`.

## Conservative mode (no recent TS pass)

If running this operation standalone without a known signal list:

- Only convert identifiers that are clearly class fields. Skip anything you cannot identify.
- For ambiguous cases, leave a comment:
  `<!-- TODO(ng-migrate): signals-template: cannot determine if 'foo' is a signal — manual review -->`

## Formatting

- Keep whitespace, line breaks, indentation, and attribute ordering identical.
- Only the changed identifiers should differ from the original.

## Validation

- No new HTML elements added or removed.
- No bindings/events/pipes/i18n attributes lost.
- Method calls untouched (`(click)="doThing()"` stays the same).
- Every previously-plain read of a signal-backed property is now a call.
