# Operation: template-control-flow

Migrate Angular templates from legacy structural directives to the new built-in control flow syntax (Angular 17+).

- `*ngIf` → `@if` / `@else`
- `*ngFor` → `@for` (with `track`)
- `[ngSwitch]` / `*ngSwitchCase` / `*ngSwitchDefault` → `@switch` / `@case` / `@default`
- `@defer` — **only** introduce when the user explicitly marks a section as deferrable. Default: do not add.

## Hard constraints

1. Syntax migration only — no behavior changes, no logic refactors.
2. Preserve DOM structure and CSS classes as closely as possible.
3. Preserve every binding, pipe, i18n, aria attribute, event handler, template reference (`#tpl`), and local template variable.
4. Do not change component TS code, inputs/outputs, or rename anything.
5. If something cannot be migrated safely, leave it as-is and add a short HTML comment explaining why:
   `<!-- TODO(ng-migrate): control-flow: <reason> -->`

## A) `*ngIf` → `@if`

### Simple
```html
<div *ngIf="cond">...</div>
```
→
```html
@if (cond) { <div>...</div> }
```

### With `else`
```html
<ng-container *ngIf="cond; else elseTpl">A</ng-container>
<ng-template #elseTpl>B</ng-template>
```
→
```html
@if (cond) { A } @else { B }
```

- If the `<ng-template>` is **only** used as the else branch, inline it (and remove the now-unused `<ng-template>`).
- If it is referenced elsewhere (`<ng-container *ngTemplateOutlet="elseTpl">` etc.), keep the `<ng-template>` and reference it from `@else { <ng-container *ngTemplateOutlet="elseTpl"></ng-container> }`.

### `else if` chains via nested templates
Convert nested `*ngIf; else nextTpl` chains into `@if / @else if / @else` blocks. Inline the templates that are only used in the chain.

### With `as` alias
```html
<div *ngIf="getUser() as user">{{ user.name }}</div>
```
→
```html
@if (getUser(); as user) { <div>{{ user.name }}</div> }
```

### `<ng-container *ngIf>` wrapping
Prefer migrating to `@if` without introducing a new DOM element. If a wrapper element is needed for layout/CSS, keep the original element inside the block.

## B) `*ngFor` → `@for`

`@for` requires `track`. If the legacy code has no `trackBy`, default to `track $index`.

### Basic
```html
<li *ngFor="let item of items">{{ item }}</li>
```
→
```html
@for (item of items; track $index) { <li>{{ item }}</li> }
```

### With `trackBy: trackByFn`
```html
<li *ngFor="let item of items; trackBy: trackByFn">...</li>
```
→
```html
@for (item of items; track trackByFn($index, item)) { <li>...</li> }
```

If the trackBy function clearly returns `item.id` (single line, easy to inline), prefer:
```html
@for (item of items; track item.id) { ... }
```
Otherwise keep the `track trackByFn($index, item)` form.

### With local variables
```html
<li *ngFor="
  let item of items;
  index as i;
  first as isFirst;
  last as isLast;
  even as isEven;
  odd as isOdd;
  count as c
">...</li>
```
→
```html
@for (
  item of items;
  track $index;
  let i = $index;
  let isFirst = $first;
  let isLast = $last;
  let isEven = $even;
  let isOdd = $odd;
  let c = $count
) { <li>...</li> }
```

### With `@empty`
If there is an explicit "no items" UI immediately near the loop (e.g. a sibling `*ngIf="!items.length"` or `*ngIf="items.length === 0"`), fold it into:
```html
@for (item of items; track $index) { ... } @empty { ...empty UI... }
```
Only do this when the empty UI is unambiguously paired with the loop. Otherwise leave the empty-state block as-is.

## C) `[ngSwitch]` → `@switch`

```html
<div [ngSwitch]="value">
  <span *ngSwitchCase="'A'">A</span>
  <span *ngSwitchCase="'B'">B</span>
  <span *ngSwitchDefault>D</span>
</div>
```
→
```html
@switch (value) {
  @case ('A') { <span>A</span> }
  @case ('B') { <span>B</span> }
  @default { <span>D</span> }
}
```

The wrapping `<div [ngSwitch]>` is removed unless it has other bindings/classes that need to stay — in which case keep the `<div>` (without `[ngSwitch]`) and put `@switch` inside it.

## D) `@defer` — opt-in only

Do not introduce `@defer` automatically. Only when the user explicitly says "defer this block" or marks a region with a comment like `<!-- defer -->`. When introducing, preserve any existing placeholder/loading/error scaffolding:

```html
@defer { ... }
@placeholder { ... }
@loading { ... }
@error { ... }
```

## Keep as-is (do NOT migrate)

These are not part of this operation:

- `*ngTemplateOutlet`, `*ngComponentOutlet`
- Third-party / library structural directives: `*matRowDef`, `*matHeaderRowDef`, `*cdkVirtualFor`, `*ngrxLet`, `*rxFor`, etc.
- Custom structural directives from the project that are not the four built-ins above.

If you encounter one and it interacts with surrounding `*ngIf`/`*ngFor`, migrate the built-ins around it and leave the third-party directive untouched. Add a comment if the surrounding migration could affect it.

## Output

- Edit the `.html` file in place.
- No new DOM wrappers introduced unless unavoidable.
- All bindings/events/pipes/i18n/aria attributes preserved.
- All template reference variables (`#tpl`) still work.
- The result compiles under Angular 17/18+ template syntax.
