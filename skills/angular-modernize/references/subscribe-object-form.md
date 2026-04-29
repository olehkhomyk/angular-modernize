# subscribe-object-form

Convert positional `subscribe(next, error?, complete?)` → observer object `subscribe({ next, error, complete })`.

## Rules

Mechanical only. No logic, operator, formatting, or `this`-binding changes. Preserve parameter names. Keep arrow vs `function` forms exactly as they were.

| Before                                          | After                                                                |
| ----------------------------------------------- | -------------------------------------------------------------------- |
| `.subscribe(next, error)`                       | `.subscribe({ next, error })`                                        |
| `.subscribe(next, error, complete)`             | `.subscribe({ next, error, complete })`                              |
| `.subscribe(next)`                              | `.subscribe({ next })` (don't strip to bare `.subscribe()`)          |
| `.subscribe(handleNext, handleError)`           | `.subscribe({ next: handleNext, error: handleError })`               |
| `.subscribe(function(v){...}, function(e){...})` | same shape, just wrapped — keep `function`s as-is                   |

`next`/`error`/`complete` keys map by position. Inline arrow lambdas become the property values verbatim.

## Skip

- `subscribe(...)` on `EventEmitter` or other non-RxJS sources. Leave + TODO: `// TODO(ng-migrate): subscribe-object-form: non-RxJS subscribe — manual review`.
- Anything inside `pipe(...)` is untouched.
