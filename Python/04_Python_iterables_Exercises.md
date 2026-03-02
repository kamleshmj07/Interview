# Python Iterables, Iterators & Generators — Practical Exercises for Senior Developers

---

## ITERABLES

---

### Exercise 1 — Objects with `__iter__()` and `__getitem__()`

**Context:** You're building a time-series data store. Records are stored internally in a fixed-size ring buffer. The store must be iterable both by protocol (`__iter__`) and by legacy index access (`__getitem__`), and must play nicely with all standard Python iteration tooling.

**Tasks:**
1. Implement `class TimeSeriesStore` backed by a plain list internally. Implement `__iter__()` returning a fresh iterator over the stored records each time.
2. Also implement `__getitem__(index)` supporting both integer and slice access.
3. Verify that `for record in store` works, `list(store)` works, `"record" in store` works (via `__contains__` fallback through `__iter__`), and `zip(store, store)` produces pairs correctly.
4. Implement a second class `LegacyStore` that has **only** `__getitem__` (no `__iter__`) and show that Python's iteration protocol still works via the `__getitem__` fallback — starting at index 0, stopping at `IndexError`.
5. Show the difference: `iter(store)` on a `TimeSeriesStore` returns the same type every time; `iter(legacy)` returns a `iterator` object created by Python automatically.
6. Confirm `TimeSeriesStore` is **not** an iterator itself — calling `next(store)` raises `TypeError`.

**Constraint:** Do not subclass `list`, `tuple`, or any built-in iterable.

---

### Exercise 2 — `for` Loop Mechanics & `iter()` / `next()`

**Context:** A query result paginator that fetches pages lazily — you need to understand exactly what Python's `for` loop does under the hood before building it.

**Tasks:**
1. Manually replicate the `for x in iterable` loop using only `iter()` and `next()` with a `try/except StopIteration` block — for a list, a string, and a `range` object.
2. Show that calling `iter()` on a list returns a `list_iterator`, and that calling `iter()` again on the **same list** returns a **new**, independent iterator (reset to position 0).
3. Show that calling `iter()` on a `list_iterator` returns **itself** — `iter(it) is it`.
4. Implement `my_for(iterable, fn)` — replicates `for x in iterable: fn(x)` without using any `for` or `while` with built-in iteration, only `iter`/`next`/`StopIteration`.
5. Implement `my_zip(*iterables)` — produces tuples from multiple iterables, stopping at the shortest, using only `iter` and `next`.
6. Implement `my_enumerate(iterable, start=0)` using only `iter` and `next`.

---

### Exercise 3 — Iterable Exhaustion & Re-iteration

**Context:** A data pipeline that reads a CSV file once and must route values to multiple downstream consumers — understanding exhaustion is critical to avoid silent data loss.

**Tasks:**
1. Create a `list` and a `list_iterator` from it. Consume the iterator halfway. Show that iterating over the **list** again produces all items, but iterating over the **iterator** again produces only the remaining items.
2. Demonstrate exhaustion: fully consume a `list_iterator`, then show the `for` loop over it produces nothing and `next()` raises `StopIteration` immediately.
3. Implement `class ReusableIterator` — wraps any iterable and returns a fresh iterator each time `__iter__` is called (stores the source iterable, not the iterator).
4. Implement `class SinglePassIterator` — wraps any iterable, raises `AlreadyExhaustedError` on a second iteration attempt rather than silently returning nothing.
5. Show the classic mistake: passing an iterator (not an iterable) to a function that iterates twice — write `count_and_list(iterable)` that calls `len(list(iterable))` then `list(iterable)` — show the second call returns `[]` when given an iterator, and fix it.
6. Use `itertools.tee(iterable, n)` to safely fork a single iterator into two independent copies — demonstrate both can be consumed independently.

---

## ITERATORS

---

### Exercise 4 — Implementing `__iter__` and `__next__`

**Context:** A database cursor simulation that streams rows from a large result set one at a time, with support for peeking at the next row without consuming it.

**Tasks:**
1. Implement `class DBCursor` with:
   - `__init__(self, rows: list)` — stores rows and sets position to 0.
   - `__iter__(self)` returning `self`.
   - `__next__(self)` advancing position and returning the next row; raising `StopIteration` when exhausted.
2. Show that `iter(cursor) is cursor` — the iterator protocol contract.
3. Add a `peek()` method that returns the next item **without** advancing the cursor. Raises `StopIteration` if exhausted.
4. Add a `reset()` method that resets the cursor to position 0.
5. Implement `class LimitedCursor(DBCursor)` that accepts a `limit: int` and raises `StopIteration` after `limit` rows regardless of how many rows remain.
6. Show that once a `DBCursor` is exhausted, resetting it and iterating again produces all rows from the beginning — unlike a true one-pass iterator.
7. Implement `class InfiniteCounter(start=0, step=1)` — an infinite iterator with no `StopIteration`. Show that using it in a bare `for` loop without a `break` or `islice` hangs — then demonstrate the safe pattern using `itertools.islice`.

---

### Exercise 5 — Internal State & One-Pass Traversal

**Context:** A stateful tokeniser that processes a stream of characters and maintains parser state between `next()` calls.

**Tasks:**
1. Implement `class Tokenizer` — takes a string `source` and yields tokens (words, numbers, punctuation) one at a time via `__next__`. Internal state: `_pos`, `_source`, `_length`.
2. Show that two `Tokenizer` instances on the same string are **independent** — advancing one does not affect the other.
3. Show that a `Tokenizer` is **one-pass**: once exhausted, creating a new iterator from it returns `self` (already exhausted), unlike a list.
4. Implement `class ZipIterator` — takes two iterators and yields `(a, b)` pairs from both simultaneously. Stops when either is exhausted. Uses only `__iter__` and `__next__` internally — no `zip`, no list conversion.
5. Implement `class ChainIterator` — takes any number of iterables and iterates through each in sequence without materialising them into a list. Track which source is currently active as internal state.
6. Verify that all your iterators satisfy the iterator protocol: `iter(it) is it` must be `True` for each.

---

### Exercise 6 — `StopIteration` & Exhaustion Patterns

**Context:** A message queue consumer that must handle empty queues, partial reads, and sentinel values gracefully.

**Tasks:**
1. Implement `safe_next(iterator, default=None)` — returns `default` instead of raising when the iterator is exhausted (mirrors `next(it, default)` built-in — implement it manually, then compare with the built-in).
2. Implement `class SentinelIterator` — wraps an iterator and stops when it encounters a specific sentinel value (e.g., `None` or `"STOP"`), treating it as end-of-stream rather than yielding it.
3. Demonstrate that `StopIteration` raised **inside a generator** has different semantics than inside `__next__` — inside a generator body it becomes `RuntimeError: generator raised StopIteration` (PEP 479). Write the code that shows this, and the fix.
4. Implement `class BufferedIterator` — reads ahead `n` items into a buffer. Exposes `has_next() -> bool` without consuming. Raises `StopIteration` correctly when both buffer and source are empty.
5. Show the classic `for`/`StopIteration` interaction: a `StopIteration` raised inside the body of a `for` loop (not from `__next__`) **silently terminates** the loop in Python < 3.7 — demonstrate the behaviour and explain why PEP 479 fixed this.
6. Implement `first_n(iterable, n)` and `skip_n(iterable, n)` using only `iter`/`next` without `itertools` or slicing.

---

## GENERATORS

---

### Exercise 7 — Generator Functions & `yield`

**Context:** A financial data feed that streams large time-bucketed datasets — memory efficiency is mandatory.

**Tasks:**
1. Implement `def read_transactions(filepath: str)` as a generator — opens a CSV, yields one parsed `dict` per line, never loading the entire file into memory. Simulate with an in-memory list.
2. Implement `def running_total(amounts)` — a generator that takes an iterable of numbers and yields the cumulative sum after each item.
3. Implement `def sliding_window(iterable, n: int)` — yields tuples of `n` consecutive items: `(1,2,3), (2,3,4), (3,4,5)` etc.
4. Implement `def interleave(*iterables)` — yields one item from each iterable in round-robin order until all are exhausted (unequal lengths: skip exhausted sources, continue with remaining).
5. Show that the generator function body does **not execute** until the first `next()` call — insert a `print("starting")` before the first `yield` and prove it only prints when iteration begins.
6. Show the generator's suspended state: inspect `gen.gi_frame`, `gen.gi_running`, and `gen.gi_code` between `next()` calls. Show `gi_frame` is `None` after exhaustion.
7. Show that a generator is both an iterator and an iterable: `iter(gen) is gen` is `True`.

---

### Exercise 8 — Generator Expressions

**Context:** A log analysis pipeline processing millions of lines — generator expressions must replace list comprehensions throughout.

**Tasks:**
1. Rewrite each of the following list comprehensions as a generator expression and prove they produce identical results:
   - `[x**2 for x in range(1_000_000)]`
   - `[line.strip() for line in lines if line.startswith("ERROR")]`
   - `[int(x) for x in tokens if x.isdigit()]`
2. Use `sys.getsizeof()` to compare the size of a list comprehension vs. a generator expression for `range(1_000_000)` — show the dramatic memory difference.
3. Chain generator expressions without materialising intermediates:
   ```python
   lines = (line.strip() for line in raw_lines)
   errors = (l for l in lines if "ERROR" in l)
   messages = (l.split("|")[2] for l in errors)
   ```
   Show the entire pipeline is lazy — no item is processed until `next()` is called on `messages`.
4. Show the **scope trap**: a generator expression captures the **loop variable by reference from the enclosing scope**, while a list comprehension has its own scope in Python 3. Demonstrate and explain.
5. Implement `pipeline(*fns)` using generator expressions — takes a seed iterable and a sequence of single-argument transform functions, chains them lazily.
6. Show that passing a generator expression to `sum()`, `max()`, `min()`, `any()`, `all()` is idiomatic and efficient — demonstrate short-circuit behaviour with `any()`.

---

### Exercise 9 — `yield from` & Sub-generator Delegation

**Context:** A recursive file system walker and a coroutine pipeline with transparent delegation.

**Tasks:**
1. Implement `def flatten(nested)` — recursively flattens arbitrarily nested lists using `yield from`. Compare with an explicit `for` loop version and a stack-based version.
2. Implement `def walk_tree(node: dict)` — a tree is `{"value": x, "children": [...]}`. Use `yield from` to recursively yield all values depth-first.
3. Show that `yield from iterable` is equivalent to `for item in iterable: yield item` for simple iteration — then show the **non-equivalence** for coroutines: `yield from` transparently passes `send()` values and `throw()` exceptions into the sub-generator; a manual loop cannot.
4. Implement `def chain_generators(*gens)` using `yield from` — replaces `itertools.chain`.
5. Implement a two-level coroutine pipeline: `producer()` yields data items, `accumulator()` uses `yield from producer()` and accumulates a running sum, returning the final total via `StopIteration.value`. Show how `yield from` captures the return value.
6. Show that the return value of a generator (the value after `return`) is carried in `StopIteration.value` — access it both manually and via `yield from`.

---

### Exercise 10 — `send()`, `throw()`, and `close()`

**Context:** A stateful coroutine pipeline for real-time stream processing, where the consumer drives the producer.

**Tasks:**

**`send()`:**
1. Implement `def running_average()` — a coroutine that receives numbers via `send(value)` and yields the current running average after each. Remember: the first `next()` (or `send(None)`) advances to the first `yield` before any value can be sent.
2. Implement a `@coroutine` decorator that auto-primes (calls `next()` once) so callers don't need to prime manually.
3. Implement `def echo_transformer(transform_fn)` — receives values via `send()`, applies `transform_fn`, and yields the result. Chain two together: `upper = echo_transformer(str.upper)` → `trimmer = echo_transformer(str.strip)`.

**`throw()`:**
4. Implement `def resilient_counter()` — counts upward, but when `ValueError` is thrown into it via `.throw(ValueError, "reset")`, it resets the counter to 0 and continues. When `StopIteration` is thrown, it exits cleanly.
5. Show that `throw()` delivers the exception **at the point where the generator is suspended** (inside the `yield` expression), allowing a `try/except` inside the generator to handle it.

**`close()`:**
6. Implement `def resource_generator(resource_name)` — opens a mock resource on entry, yields items, and uses `try/finally` to guarantee the resource is released when `.close()` is called mid-iteration.
7. Show that calling `.close()` causes `GeneratorExit` to be thrown into the generator at the suspension point, and that re-raising or not catching it is correct — but catching it and **not** re-raising or returning is a `RuntimeError`.

**Combining:**
8. Build `def controlled_pipeline(source)` — uses `send()` to pass control parameters (e.g., a batch size) that modify the generator's behaviour on each iteration, `throw()` to inject retry signals, and `close()` for clean shutdown.

---

## Bonus Challenge — Bring It All Together

Design a **lazy streaming data pipeline** using all three concepts: iterables, iterators, and generators.

**Requirements:**

- `class DataSource` — a reusable iterable (not an iterator) backed by a list of raw dicts. Each call to `__iter__` produces a fresh pass.
- `class TransformIterator` — a custom iterator wrapping any iterable, applying a transform function to each element lazily.
- `def validate_stream(iterable, *validators)` — a generator that yields `(record, errors)` tuples; `errors` is a list of failed validator messages. Yields every record (valid or not) — consumers decide what to do.
- `def batch(iterable, size: int)` — a generator that groups items into lists of `size` (final batch may be smaller).
- `def throttle(iterable, max_per_second: int)` — a generator that yields items but sleeps to enforce a rate limit.
- A coroutine `def sink(transform_fn)` — primed automatically via `@coroutine`, receives records via `send()`, applies `transform_fn`, accumulates results, and returns the final list via `StopIteration.value`.
- Wire everything together:

```python
source = DataSource(raw_records)

pipeline = batch(
    throttle(
        (rec for rec, errs in validate_stream(
            TransformIterator(source, normalize),
            require_fields, positive_amounts
        ) if not errs),
        max_per_second=100
    ),
    size=10
)

collector = sink(enrich_with_metadata)
for batch_records in pipeline:
    for record in batch_records:
        collector.send(record)

collector.close()
```
