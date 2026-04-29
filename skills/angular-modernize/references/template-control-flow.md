# template-control-flow

Migrate Angular 17+ built-in control flow:
- `*ngIf` → `@if` / `@else`
- `*ngFor` → `@for` (with `track`)
- `[ngSwitch]` / `*ngSwitchCase` / `*ngSwitchDefault` → `@switch` / `@case` / `@default`
- `@defer` — opt-in only.

Syntax migration only. Preserve DOM, CSS classes, all bindings, pipes, i18n, aria, events, `#tpl` refs, and local vars. Don't touch component TS.

## `*ngIf` → `@if`

```html
<div *ngIf="cond">...</div>
→ @if (cond) { <div>...</div> }
```

With `else`:
```html
<ng-container *ngIf="cond; else elseTpl">A</ng-container>
<ng-template #elseTpl>B</ng-template>
→ @if (cond) { A } @else { B }
```
- Inline the `<ng-template>` if it's only used as the else; otherwise keep it and reference it from `@else { <ng-container *ngTemplateOutlet="elseTpl" /> }`.
- `else if` chains (nested `*ngIf; else nextTpl`) → `@if / @else if / @else`.

With `as` alias:
```html
*ngIf="getUser() as user"
→ @if (getUser(); as user) { ... }
```

`<ng-container *ngIf>` wrappers — drop the `<ng-container>` and use `@if` blocks directly. Keep wrapping element only if layout/CSS needs it.

## `*ngFor` → `@for` (track required)

No `trackBy` → `track $index`.

```html
*ngFor="let item of items"
→ @for (item of items; track $index) { ... }

*ngFor="let item of items; trackBy: trackByFn"
→ @for (item of items; track trackByFn($index, item)) { ... }
```
If `trackBy` clearly returns `item.id` (one-liner), prefer `track item.id`.

Local vars:
```html
*ngFor="let item of items; index as i; first as isFirst; last as isLast; even as isEven; odd as isOdd; count as c"
→ @for (item of items; track $index;
        let i = $index; let isFirst = $first; let isLast = $last;
        let isEven = $even; let isOdd = $odd; let c = $count) { ... }
```

`@empty`: if a sibling explicitly shows "no items" (e.g. `*ngIf="!items.length"`), fold into:
```html
@for (...) { ... } @empty { ...empty UI... }
```
Only when the empty UI is unambiguously paired with the loop.

## `[ngSwitch]` → `@switch`

```html
<div [ngSwitch]="value">
  <span *ngSwitchCase="'A'">A</span>
  <span *ngSwitchDefault>D</span>
</div>
→
@switch (value) {
  @case ('A') { <span>A</span> }
  @default { <span>D</span> }
}
```
Drop the `[ngSwitch]` wrapper unless it has other bindings/classes. If kept, put `@switch` inside it.

## `@defer` (opt-in only)

Do NOT add automatically. Only when the user marks a region (`<!-- defer -->` or explicit ask). Preserve existing placeholder/loading/error blocks if present:
```html
@defer { ... } @placeholder { ... } @loading { ... } @error { ... }
```

## Skip (do not migrate)

- `*ngTemplateOutlet`, `*ngComponentOutlet`
- Third-party / library structural directives (`*matRowDef`, `*cdkVirtualFor`, `*ngrxLet`, `*rxFor`, custom project directives)

If a third-party directive interacts with a surrounding `*ngIf`/`*ngFor` you're migrating, migrate the built-ins around it and add `<!-- TODO(ng-migrate): control-flow: third-party directive nearby — verify -->` if the change could affect it.

## Output

- Edit the `.html` (or inline template) in place.
- No new DOM wrappers unless unavoidable.
- All bindings/events/pipes/i18n/aria preserved.
- `#tpl` refs still work.
