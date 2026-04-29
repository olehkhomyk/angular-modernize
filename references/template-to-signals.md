# Operation: signals-template (HTML reads → call form)

After a TS file has been migrated to signals, its template needs every read of those signals to be a function call (`property` → `property()`). This pass is **read-only** — do not introduce `.set()` or `.update()` calls in templates.

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

## Do NOT

- Add `()` to method calls — methods already use `()`. Only convert plain identifier reads.
- Add `()` to event handlers like `(click)="doThing()"` — those are method calls.
- Convert reads inside event handler **arguments** unless the identifier is a signal: `(click)="save(prop)"` → `(click)="save(prop())"` only when `prop` is a signal.
- Add `.set(...)` / `.update(...)` calls to the template (banana-in-a-box `[(ngModel)]` and similar should be left alone — it's a different migration).
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
