# Python Decorators — Practical Exercises for Senior Developers

---

### Exercise 1 — Basic Decorator Structure

**Context:** You're building an API server. Before any handler function runs, you need a uniform "health-check" layer that prints the function name on entry and exit — without modifying a single handler.

**Tasks:**
1. Implement a `trace` decorator using the full three-part structure: outer function, inner `wrapper`, and `return wrapper`.
2. Apply it manually **without** `@` syntax: `traced_handler = trace(handle_request)` — then call `traced_handler("GET", "/users")`.
3. Define three handler functions: `handle_get`, `handle_post`, `handle_delete` — each prints its method and returns a mock response dict.
4. Show that the decorator does **not** mutate the original function — calling `handle_get` directly still works undecorated.
5. Inspect `wrapper.__closure__` to show that `fn` (the original function) is captured inside the closure.
6. Add a call counter inside `trace` using a mutable container (`calls = [0]`) — increment on each wrapper invocation and expose it as `wrapper.call_count`.

**Constraint:** No `functools.wraps`, no `@` syntax, no classes — pure function-based structure only.

---

### Exercise 2 — `@` Syntax

**Context:** A background job runner that needs consistent pre/post logging around every job function.

**Tasks:**
1. Re-implement the `trace` decorator from Exercise 1, this time applying it with `@trace` syntax on four job functions: `sync_users`, `send_emails`, `generate_report`, `archive_logs`.
2. Show that `@trace` is exactly equivalent to `sync_users = trace(sync_users)` — prove it by checking `id()` before and after manual application matches the `@` result.
3. Apply a second decorator `@mark_as_job` (sets `fn.is_job = True`) and show the stacking: `@mark_as_job` on top of `@trace`.
4. Write a `job_registry` that scans a list of functions and collects only those with `is_job == True`.
5. Demonstrate that `@` is just syntactic sugar — write a function that applies a list of decorators programmatically (equivalent to stacking `@`s) using `functools.reduce`.

---

### Exercise 3 — Decorators with `*args` and `**kwargs`

**Context:** A data ingestion service where functions have wildly different signatures — the decorator must be fully transparent to all of them.

**Tasks:**
1. Implement a `@log_call` decorator that logs the function name, all positional args, all keyword args, and the return value — works on any function signature.
2. Apply `@log_call` to:
   - `def ingest(source: str, batch_size: int = 100) -> int`
   - `def transform(data: list, *, schema: str, strict: bool = False) -> dict`
   - `def export(dest: str, *formats, overwrite: bool = False) -> bool`
3. Show that calling each with its full, partial, or mixed arguments logs correctly.
4. Implement `@enforce_positional_only(n)` — a decorator that raises `TypeError` if fewer than `n` positional arguments are passed to the wrapped function (inspect `args` in the wrapper).
5. Implement `@forbid_kwargs(*names)` — raises `TypeError` if any of the listed keyword argument names appear in the call.
6. Show that `wrapper(*args, **kwargs)` without modification is the only safe pattern — demonstrate what breaks when the wrapper hard-codes a specific signature.

---

### Exercise 4 — Preserving Function Metadata (`functools.wraps`)

**Context:** A decorator library consumed by a large team — bad metadata breaks `help()`, auto-generated docs, and introspection tooling.

**Tasks:**
1. Write a `@sanitize` decorator **without** `functools.wraps`. Apply it to a richly documented function:
   ```python
   def process_payment(amount: float, currency: str = "USD") -> dict:
       """Process a payment transaction.
       
       Args:
           amount: Transaction amount in base units.
           currency: ISO 4217 currency code.
       Returns:
           Transaction receipt as a dict.
       """
   ```
   Show that `__name__`, `__doc__`, `__annotations__`, `__qualname__`, and `__module__` are all wrong after decoration.

2. Fix `@sanitize` with `functools.wraps`. Verify all five attributes are restored.
3. Show that `functools.wraps` also copies `__dict__` — attach `process_payment.team = "payments"` and confirm it survives decoration.
4. Show that `functools.wraps` sets `__wrapped__` — write `unwrap(fn)` that follows `__wrapped__` until it reaches the original.
5. Show that `help(process_payment)` renders the correct docstring after `functools.wraps`.
6. Demonstrate that `inspect.signature(decorated)` returns the **original** signature (parameter names + annotations) — not `(*args, **kwargs)`.

---

### Exercise 5 — Stacked / Multiple Decorators

**Context:** A microservice endpoint that needs auditing, rate-limiting, and input validation — each concern is a separate decorator.

**Tasks:**
1. Implement three decorators:
   - `@audit` — logs `[AUDIT] fn_name called by {user}` (reads a `user` kwarg or defaults to `"anonymous"`)
   - `@rate_limit(max_calls, period)` — raises `RateLimitError` after `max_calls` invocations within `period` seconds
   - `@require_auth` — raises `PermissionError` if the kwarg `token` is missing or falsy
2. Stack them on `get_dashboard(user: str, token: str) -> dict` in the order: `@audit` → `@rate_limit(5, 60)` → `@require_auth` (top to bottom).
3. Show and explain the execution order: which decorator's wrapper runs first when `get_dashboard(...)` is called? Draw the call stack in comments.
4. Swap `@audit` and `@require_auth`, re-run without a token, and show how the error message changes — proving order affects observable behavior.
5. Inspect `get_dashboard.__wrapped__.__wrapped__` to navigate the wrapper chain manually.
6. Write a `decorator_stack(*decorators)` utility that applies a list of decorators in the correct top-to-bottom order, equivalent to stacking `@`s.

---

### Exercise 6 — Class Decorators (Decorating Classes)

**Context:** A plugin framework where every registered plugin class must be auto-validated, timestamped, and catalogued.

**Tasks:**
1. Implement `@register_plugin` — a function-based class decorator that:
   - Adds a `registered_at: datetime` class attribute.
   - Adds the class to a module-level `PLUGIN_REGISTRY: dict[str, type]` keyed by class name.
   - Raises `TypeError` if the class doesn't define a `run(self)` method.
2. Apply it to `AudioPlugin`, `VideoPlugin`, and `NetworkPlugin` (each implementing `run()`).
3. Show that the class itself is unchanged (same `id`) — `@register_plugin` mutates and returns the original class, not a wrapper.
4. Implement `@freeze` — a class decorator that overrides `__setattr__` to raise `FrozenError` after `__init__` completes (use a flag stored in the instance's `__dict__` directly via `object.__setattr__`).
5. Implement `@add_repr` — a class decorator that injects a `__repr__` method based on the class's `__init__` signature, outputting `ClassName(attr=value, ...)` for each parameter.
6. Demonstrate that class decorators compose just like function decorators: `@add_repr` + `@freeze` + `@register_plugin` on a single class.

---

### Exercise 7 — Classes as Decorators

**Context:** A stateful request throttler where the decorator itself must maintain per-endpoint call history across invocations.

**Tasks:**
1. Implement `class Throttle` as a decorator:
   - `__init__(self, fn)` stores the function and initializes a call log.
   - `__call__(self, *args, **kwargs)` logs the call timestamp and invokes the function.
   - Add a `stats()` method returning total calls, first call time, and last call time.
2. Apply `@Throttle` to `fetch_prices`, `fetch_orders`, and `fetch_inventory`.
3. Show that `fetch_prices` is now an **instance of `Throttle`**, not a function — `isinstance(fetch_prices, Throttle)` is `True`.
4. Show the problem: `fetch_prices.__name__` doesn't exist natively on the instance. Fix it by setting `self.__name__` and `self.__doc__` in `__init__`.
5. Implement `class ThrottleWithLimit` that subclasses `Throttle`, adds a `max_calls` limit, and raises `ThrottleError` when exceeded. Re-apply as `@ThrottleWithLimit(max_calls=3)` — note this requires `__init__` to accept `max_calls` and `__call__` to **return a decorator**, not directly call `fn`. Implement this two-phase `__call__` pattern.
6. Compare `class Throttle` vs. a closure-based throttle — list two advantages and two disadvantages of each approach.

---

### Exercise 8 — Decorator Factories (Decorators Taking Parameters)

**Context:** A configurable validation and transformation layer for a REST API.

**Tasks:**
1. Implement `@validate(schema: dict)` — a decorator factory. `schema` maps parameter names to `(type, validator_fn)` tuples. At call time, checks each argument against its type and validator; raises `ValidationError` with a detailed message on failure.
2. Implement `@transform_output(fn: callable)` — applies `fn` to the return value before returning it to the caller.
3. Implement `@deprecated(reason: str, replacement: str = None)` — prints a `DeprecationWarning` with the reason and optional replacement name every time the decorated function is called.
4. Implement `@retry(max_attempts=3, delay=0.0, exceptions=(Exception,), on_failure=None)` — full-featured retry factory. `on_failure` is an optional callback called with `(fn, attempts, last_exception)` if all retries fail.
5. Chain `@validate`, `@transform_output`, and `@retry` on a single function and show all three are active simultaneously.
6. Show the factory pattern's structure clearly: the **three levels** of nesting — `factory(params)` → `decorator(fn)` → `wrapper(*args, **kwargs)` — and label each level in comments.

---

### Exercise 9 — Common Use Cases

**Context:** A production service requiring timing, structured logging, caching, and access control on its core functions.

**Tasks:**

**Timing:**
1. Implement `@timer` — measures wall-clock time using `time.perf_counter()`, stores the last duration as `fn.last_duration`, and optionally prints if `verbose=True` (make it a factory: `@timer(verbose=True)`).

**Logging:**
2. Implement `@structured_log` — emits a JSON-formatted log line with `fn_name`, `args`, `kwargs`, `result`, `duration_ms`, and `timestamp`. Use `functools.wraps`. Make severity configurable: `@structured_log(level="INFO")`.

**Caching:**
3. Implement `@memoize` — caches return values by `(args, frozenset(kwargs.items()))` key. Expose `fn.cache_info()` returning `{"hits": n, "misses": n, "size": n}` and `fn.cache_clear()`. Show it works on `fibonacci(n)` and `expensive_query(id, filters=None)`.

**Authentication:**
4. Implement `@require_roles(*roles)` — inspects a `current_user: dict` kwarg (must have a `"roles"` key). Raises `AuthorizationError` if the user holds none of the required roles. Show it working with `@require_roles("admin", "superuser")`.

**Combining:**
5. Apply all four decorators to a single `get_report(report_id: int, current_user: dict) -> dict` function in a sensible stacking order. Justify the order in comments.

---

### Exercise 10 — Attribute Preservation

**Context:** A framework that introspects decorated functions at startup to build routing tables, API schemas, and documentation — all metadata must survive decoration.

**Tasks:**
1. Create a richly annotated function `process_order` with: a multi-paragraph docstring, full type annotations, custom attributes (`process_order.version`, `process_order.tags`, `process_order.owner`), and a default argument.
2. Apply three decorators sequentially (using stacking). After each layer, record the values of: `__name__`, `__doc__`, `__annotations__`, `__qualname__`, `__module__`, `__dict__`, `__wrapped__`.
3. Show the difference when one decorator in the stack is missing `functools.wraps` — which layer breaks the chain, and how does `__wrapped__` help diagnose it?
4. Implement `@preserve_all` — a decorator that explicitly copies all of `WRAPPER_ASSIGNMENTS` **and** `WRAPPER_UPDATES`, plus any extra keys found in `fn.__dict__`, to the wrapper.
5. Write an `assert_metadata_intact(original_fn, decorated_fn)` helper that checks all critical attributes match and raises `AssertionError` with a diff if any are missing or wrong.
6. Use `inspect.signature()` to confirm parameter names, defaults, and annotations survive all three decoration layers.
7. Demonstrate that `functools.wraps` does **not** fix `inspect.signature` returning `(*args, **kwargs)` automatically — this requires `__wrapped__` to be set so `inspect.signature` can follow it. Show both the broken and correct behavior.

---

## Bonus Challenge — Bring It All Together

Design `@pipeline_decorator` — a meta-decorator factory that composes multiple decorators into a single reusable decorator, with full metadata preservation and introspection support.

**Requirements:**

- `@pipeline_decorator(*decorators)` returns a single decorator that applies all `decorators` in the correct order (as if they were stacked top-to-bottom).
- The resulting decorator preserves `__name__`, `__doc__`, `__annotations__`, `__dict__`, and `__wrapped__` correctly.
- Supports an optional `label: str` parameter used in log output: `@pipeline_decorator(*decorators, label="api_endpoint")`.
- Implement `DecoratorRegistry` as a **class decorator factory** that:
  - Tracks every function decorated with any decorator from a given registry instance.
  - Exposes `registry.list_decorated()` returning a list of `(fn_name, decorators_applied)` tuples.
  - Supports `registry.audit(fn_name)` returning full metadata of a registered function.
- All wrappers must use `functools.wraps`.
- The `__wrapped__` chain must be navigable all the way to the original function.

You should be able to write:

```python
registry = DecoratorRegistry()

api = pipeline_decorator(
    registry.track,
    require_auth,
    rate_limit(10, 60),
    structured_log(level="INFO"),
    timer(verbose=False),
    label="api_endpoint",
)

@api
def get_user(user_id: int, token: str) -> dict:
    """Fetch a user by ID."""
    return {"id": user_id}

print(get_user.__name__)            # get_user
print(get_user.__doc__)             # Fetch a user by ID.
print(registry.list_decorated())    # [('get_user', ['track', 'require_auth', ...])]
print(registry.audit("get_user"))   # full metadata dict
```
