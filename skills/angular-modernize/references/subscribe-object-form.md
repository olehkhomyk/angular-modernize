# Operation: subscribe-object-form

Convert deprecated positional `subscribe(nextFn, errorFn?, completeFn?)` calls to the modern observer-object form `subscribe({ next, error, complete })`.

## Hard constraints

1. No behavior changes. No logic refactors.
2. Do **not** modify the `pipe(...)` chain, operators, scheduling, or order of statements.
3. Keep variable names, indentation, and structure the same.
4. Do **not** introduce new helpers, classes, or abstractions.
5. Preserve `this`-binding: do **not** convert arrow functions to regular `function` or vice versa.

## Conversions

### A) `next` + `error`

```ts
.subscribe(
  (value) => { ... },
  (err) => { ... }
);
```
→
```ts
.subscribe({
  next: (value) => { ... },
  error: (err) => { ... }
});
```

### B) `next` + `error` + `complete`

```ts
.subscribe(
  (value) => { ... },
  (err) => { ... },
  () => { ... }
);
```
→
```ts
.subscribe({
  next: (value) => { ... },
  error: (err) => { ... },
  complete: () => { ... }
});
```

### C) Only `next`

```ts
.subscribe((value) => { ... });
```
→
```ts
.subscribe({ next: (value) => { ... } });
```

(Smallest change — keep the wrapping. Do not strip arguments to a bare `.subscribe()`.)

### D) Named function references

```ts
.subscribe(handleNext, handleError);
```
→
```ts
.subscribe({ next: handleNext, error: handleError });
```

### E) Regular `function() { ... }` (preserves `this`)

```ts
.subscribe(function (res) { ... }, function (err) { ... });
```
→
```ts
.subscribe({
  next: function (res) { ... },
  error: function (err) { ... }
});
```

Do not convert these to arrow functions. The `function` form may be intentional for `this` binding.

## Do NOT touch

- `subscribe(...)` calls on `EventEmitter` or other non-RxJS sources. If unsure, leave it and add a comment:
  `// TODO(ng-migrate): subscribe-object-form: non-RxJS subscribe — manual review`
- The Observable chain itself.
- Parameter names — preserve them exactly (e.g. if the error handler is named `httpError`, keep that).

## Validation checklist

- Same execution order.
- No operator changes.
- Same error/completion behavior.
- No `this`-binding regressions.
- No accidental whitespace/format churn outside the migrated call.
