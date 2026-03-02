# Python First-Class & Higher-Order Functions — Practical Exercises for Senior Developers

---

## FIRST-CLASS FUNCTIONS

---

### Exercise 1 — Functions as Objects

**Context:** You're building an audit system that needs to introspect callable units at runtime — inspecting their metadata, type identity, and bytecode properties.

**Tasks:**
1. Define three functions: `validate_email(email: str) -> bool`, `validate_phone(phone: str) -> bool`, and `validate_age(age: int) -> bool`.
2. Confirm that `type(validate_email)` is `type(lambda: None)` — prove all functions share the same type.
3. Access and print: `__name__`, `__qualname__`, `__doc__`, `__module__`, `__annotations__`, `__code__.co_varnames`, `__code__.co_filename`, `__code__.co_argcount`.
4. Show that functions have `__dict__` — attach custom metadata: `validate_email.version = "1.0"` and `validate_email.owner = "auth-team"`.
5. Show that `id(validate_email)` is stable across calls (functions are objects in memory, not re-created each call).
6. Use `inspect.signature()` to extract parameter names and annotations programmatically.

**Constraints:** Use real `re`-based implementations for the validators, not stubs.

---

### Exercise 2 — Assigning Functions to Variables

**Context:** A feature-flag system where the active processing strategy can be swapped at runtime by reassigning a variable.

**Tasks:**
1. Implement three text-transformation functions: `to_uppercase`, `to_titlecase`, and `to_snakecase` (converts spaces to underscores, lowercases).
2. Assign all three to a single variable `active_transform` at different points — demonstrate that the variable holds a reference, not a copy.
3. Store the original reference before reassignment, reassign, then restore — prove the original function is unchanged.
4. Show that `active_transform is to_uppercase` evaluates to `True` when assigned, then `False` after reassignment.
5. Implement a `swap_strategy(new_fn)` function that updates `active_transform` using `global` and returns the previously active function.
6. Demonstrate: call `swap_strategy` twice and prove the returned "previous" functions are the correct originals.

**Constraint:** Do not use classes or wrappers — plain function references only.

---

### Exercise 3 — Storing Functions in Data Structures

**Context:** A command dispatcher for a CLI tool — maps command names to handler functions, supports command groups (list of handlers), and priority queues (sorted handlers).

**Tasks:**
1. Build a `command_registry: dict[str, callable]` mapping string commands to handler functions (`handle_create`, `handle_delete`, `handle_list`, `handle_export`).
2. Dispatch a command by looking up and invoking: `command_registry["create"]("users")`.
3. Build a `middleware_pipeline: list[callable]` — a list of functions applied left-to-right to a string payload: `[strip_whitespace, normalize_unicode, sanitize_html, truncate_to_256]` (implement all four).
4. Execute the pipeline by iterating the list, threading the result through each function.
5. Build a `priority_handlers: list[tuple[int, callable]]`, sort by priority integer, and invoke in order, stopping at the first handler that returns a non-`None` result (chain-of-responsibility pattern).
6. Show that a `set` of functions works for deduplication: add the same function twice and verify `len == 1`.

---

### Exercise 4 — Passing Functions as Arguments

**Context:** A data-processing pipeline for a CSV ingestion service.

**Tasks:**
1. Implement `process_records(records: list[dict], validator: callable, transformer: callable, error_handler: callable) -> list[dict]` — for each record: validate it, transform it if valid, call `error_handler` if not, return processed records.
2. Implement three validators: `require_fields(record)`, `positive_amounts(record)`, `no_future_dates(record)`.
3. Implement two transformers: `normalize_currency(record)` (adds `currency: "USD"` if absent) and `enrich_with_metadata(record)` (adds `processed_at` timestamp).
4. Implement two error handlers: `log_and_skip(record, error)` and `raise_on_error(record, error)`.
5. Call `process_records` with different combinations of the above and show that behavior changes purely by swapping the function arguments.
6. Show why this pattern is superior to a long `if/elif` chain for extensibility.

---

### Exercise 5 — Returning Functions from Other Functions

**Context:** A configurable retry mechanism for network calls.

**Tasks:**
1. Implement `make_retry(max_attempts: int, delay: float, exceptions: tuple) -> callable` — returns a function `retry(fn, *args, **kwargs)` that retries `fn` up to `max_attempts` times, sleeping `delay` seconds between attempts, only catching the specified `exceptions`.
2. Implement `make_multiplier(factor: float) -> callable` — returns a function that multiplies its argument by `factor`.
3. Implement `make_validator(rules: list[callable]) -> callable` — returns a function that takes a value and returns `(bool, list[str])` where the list contains all failed rule messages.
4. Demonstrate that each call to `make_multiplier` returns a **distinct** function object: `double = make_multiplier(2)` and `triple = make_multiplier(3)` are different objects but share the same source code.
5. Show `double.__code__ is triple.__code__` — same code object, different closure state.

---

### Exercise 6 — Nested Functions (Inner Functions)

**Context:** A query-builder utility where helper logic should not pollute the module namespace.

**Tasks:**
1. Implement `build_sql_query(table: str, filters: dict, order_by: str = None, limit: int = None) -> str` using inner functions: `_build_where(filters)`, `_build_order(order_by)`, `_build_limit(limit)` — none of these should be accessible outside `build_sql_query`.
2. Show that calling `build_sql_query._build_where(...)` from outside raises `AttributeError`.
3. Implement `make_counter(start: int = 0, step: int = 1)` using an inner function `counter()` that increments a mutable container (use a list `[start]` to work around `nonlocal` restrictions in Python 2 style — then refactor to use `nonlocal`).
4. Implement a `logged_execution(fn)` function that defines an inner `wrapper(*args, **kwargs)` which logs before/after calling `fn` — without using `@` decorator syntax.
5. Show the inner function's `__qualname__` reveals its nesting: e.g., `build_sql_query.<locals>._build_where`.

---

## HIGHER-ORDER FUNCTIONS

---

### Exercise 7 — Functions Accepting Functions as Arguments

**Context:** A statistical analysis engine that applies pluggable aggregation strategies to datasets.

**Tasks:**
1. Implement `apply_aggregation(data: list[float], *aggregators: callable) -> dict[str, float]` — applies each aggregator to the data and returns a dict keyed by `aggregator.__name__`.
2. Implement aggregators: `mean`, `median`, `std_dev`, `percentile_90` — no `statistics` module, implement from scratch.
3. Implement `batch_transform(datasets: list[list], transform_fn: callable) -> list[list]` — applies `transform_fn` to every dataset.
4. Implement `conditional_apply(value, condition_fn: callable, true_fn: callable, false_fn: callable)` — a functional ternary.
5. Implement `pipe(value, *fns: callable)` — threads `value` through each function left-to-right (like Unix pipes).
6. Demonstrate `pipe(raw_data, normalize, remove_outliers, compute_zscore, round_to_2dp)` where each step is a separately defined function.

---

### Exercise 8 — Function Factories

**Context:** A configurable data-validation rule engine where rules are generated dynamically from configuration.

**Tasks:**
1. Implement `make_range_validator(min_val, max_val)` — returns a function `validator(x) -> bool` with a descriptive `__name__` and `__doc__` set on it.
2. Implement `make_regex_validator(pattern: str, flags=0)` — compiles the regex once at factory time (not per call) and returns a validator.
3. Implement `make_threshold_alert(metric_name: str, threshold: float, severity: str)` — returns a function that takes a value and returns an alert dict or `None`.
4. Implement `make_pipeline(*fns: callable)` — returns a single function that chains all `fns` in sequence.
5. Build a `rule_registry` by calling your factories with different parameters and storing the results. Show 5+ distinct validators created from 2 factory functions.
6. Prove each factory call produces a new function object (`make_range_validator(0, 10) is not make_range_validator(0, 10)`).

---

### Exercise 9 — Lambda Functions

**Context:** A reporting system requiring concise, inline transformation and sorting logic.

**Tasks:**
1. Sort a list of dicts `[{"name": "...", "score": ..., "department": "..."}]` by: (a) score descending, (b) department then name, (c) score ascending with ties broken by name — all using `sorted()` with `key=lambda ...`.
2. Use `lambda` inside `map()` to transform a list of temperatures from Celsius to Fahrenheit.
3. Use `lambda` inside `filter()` to keep only records where score > 75 and department == "Engineering".
4. Build a `dispatch_table: dict[str, callable]` mapping operation names to lambdas: `{"double": lambda x: x*2, "square": lambda x: x**2, "negate": lambda x: -x}`.
5. Show the **limits** of lambdas: (a) they cannot contain statements, (b) they cannot use `yield`, (c) they don't support annotations, (d) their `__name__` is always `"<lambda>"` — explain why this matters for debugging.
6. Refactor the lambda-heavy code from task 1–3 into named functions and compare readability. When is a lambda justified vs. harmful?

---

### Exercise 10 — Closures

**Context:** A session management system where each user session captures its own state in a closure.

**Tasks:**
1. Implement `make_session(user_id: str, permissions: list[str])` — returns a `session` function. Calling `session("check", "admin")` returns `True/False`, `session("info")` returns a dict, `session("revoke", "admin")` removes the permission. All state lives in the closure, not in any external object.
2. Implement `make_accumulator(initial: float = 0.0)` — returns `add(n)` that accumulates a running total using `nonlocal`. Show that two accumulators are independent.
3. Implement `make_memoizer()` — returns a `memoize(fn)` function. The cache dict lives in the closure. Show the cache is private — inaccessible from outside.
4. Implement `make_event_log()` — returns `(log_event, get_events, clear_events)` as a tuple of three closures sharing the same `events` list.
5. Demonstrate the **closure trap** in loops: `fns = [lambda: i for i in range(5)]` — show all return 4, explain why, then fix it with `lambda i=i: i` (default argument capture) and with `make_fn(i)` factory pattern.
6. Use `inspect.getclosurevars(fn)` to introspect what a closure has captured.

---

### Exercise 11 — `functools.wraps`

**Context:** A decorator library for a web framework — decorators must not destroy the original function's identity.

**Tasks:**
1. Implement a `@timer` decorator **without** `functools.wraps`. Show that `decorated.__name__`, `decorated.__doc__`, `decorated.__annotations__`, and `help(decorated)` are all wrong.
2. Fix `@timer` with `functools.wraps`. Show all metadata is preserved.
3. Implement `@retry(max_attempts=3, delay=0.0)` as a **parametrized decorator** using `functools.wraps`.
4. Implement `@validate_inputs(**type_map)` — a parametrized decorator that checks argument types at call time using `functools.wraps`.
5. Stack all three decorators on a single function and show that `__name__`, `__doc__`, and `__wrapped__` are correct at every layer.
6. Use `functools.wraps` `updated` parameter to also preserve `__dict__` — attach metadata to the original function and confirm it survives decoration.
7. Show that `functools.wraps` sets `__wrapped__` — use it to unwrap a decorated function back to its original with a `unwrap(fn)` utility function.

---

### Exercise 12 — `map` / `filter` / `reduce` Patterns

**Context:** A financial transaction processor.

**Tasks:**
1. Given `transactions: list[dict]` with keys `id`, `amount`, `currency`, `status`, `category`:
   - Use `map()` to convert all amounts to USD (apply a `currency_to_usd(record)` transformer).
   - Use `filter()` to keep only `status == "completed"` transactions.
   - Use `functools.reduce()` to sum all amounts into a total.
2. Chain all three into a single expression (no intermediate variables) and show the result equals the step-by-step result.
3. Rewrite the entire chain using a list comprehension + `sum()` and compare readability and performance using `timeit`.
4. Implement your own `my_map(fn, iterable)`, `my_filter(pred, iterable)`, and `my_reduce(fn, iterable, initial)` as generator-based functions (no built-ins).
5. Use `map()` with **multiple iterables**: `map(fn, list1, list2)` — implement a `weighted_sum` that maps over a values list and a weights list simultaneously.
6. Use `itertools.starmap` to apply a function to a list of argument tuples — show when it's cleaner than `map`.
7. Implement `flat_map(fn, iterable)` (like `flatMap` in Scala/JS) — maps and flattens one level.

---

## Bonus Challenge — Bring It All Together

Design a **`Pipeline` builder** — a reusable, composable data-processing system that uses every concept from this module.

Requirements:

- `Pipeline(*fns)` stores a sequence of transformation functions.
- `Pipeline.__or__(other_fn)` supports `pipeline | new_fn` syntax to append a step (returns a new `Pipeline`).
- `Pipeline.__call__(value)` executes the pipeline, threading `value` through each step.
- `Pipeline.map(fn)` returns a new `Pipeline` where the current pipeline is applied element-wise via `map()`.
- `Pipeline.filter(pred)` appends a filter step.
- `Pipeline.reduce(fn, initial)` terminates the pipeline with a reduce and returns the final scalar value.
- A `@pipeline_step` decorator factory that validates a function's single-argument signature and attaches `step_name` metadata — using `functools.wraps`.
- A `make_branching_pipeline(condition_fn, true_pipeline, false_pipeline)` function factory that routes each element to one of two pipelines based on `condition_fn`.

You should be able to write:

```python
result = (
    Pipeline(normalize, strip_whitespace)
    | remove_duplicates
    | str.lower
).map(tokenize).filter(lambda tok: len(tok) > 2).reduce(merge_tokens, initial="")
```
