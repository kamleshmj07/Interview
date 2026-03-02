# Python Iterables, Iterators & Generators — Solutions

---

## Exercise 1 — `__iter__` and `__getitem__`

```python
class TimeSeriesStore:
    def __init__(self, records=None):
        self._data = list(records or [])

    def __iter__(self):
        # Returns a FRESH iterator each call — makes this an iterable, not an iterator
        return iter(self._data)

    def __getitem__(self, index):
        return self._data[index]

    def __len__(self):
        return len(self._data)

    def append(self, record):
        self._data.append(record)


class LegacyStore:
    """Only __getitem__ — Python falls back to index-based iteration automatically."""
    def __init__(self, records):
        self._data = list(records)

    def __getitem__(self, index):
        # Python calls with 0, 1, 2... stopping at IndexError
        return self._data[index]


records = [{"ts": i, "val": i * 1.5} for i in range(5)]
store = TimeSeriesStore(records)

print(list(store))                          # all records
print({"ts": 0, "val": 0.0} in store)      # True — __contains__ via __iter__
print(list(zip(store, store)))              # zip gets two independent fresh iterators

legacy = LegacyStore(records)
for rec in legacy:                          # works via __getitem__ fallback
    print(rec["ts"], end=" ")

# iter() always returns a NEW iterator — store is not an iterator
it1 = iter(store)
it2 = iter(store)
print(it1 is it2)   # False — fresh each time

# Calling next() on the store itself fails — it is not an iterator
try:
    next(store)
except TypeError as e:
    print(f"Not an iterator: {e}")
```

---

## Exercise 2 — `for` Loop Mechanics

```python
def my_for(iterable, fn):
    it = iter(iterable)
    while True:
        try:
            item = next(it)
        except StopIteration:
            break
        fn(item)


def my_zip(*iterables):
    iterators = [iter(i) for i in iterables]
    while True:
        result = []
        for it in iterators:
            try:
                result.append(next(it))
            except StopIteration:
                return
        yield tuple(result)


def my_enumerate(iterable, start=0):
    it = iter(iterable)
    index = start
    while True:
        try:
            value = next(it)
        except StopIteration:
            return
        yield index, value
        index += 1


# iter() identity rules
lst = [1, 2, 3]
li = iter(lst)
print(iter(li) is li)          # True  — iterator returns self
print(iter(lst) is iter(lst))  # False — list always returns a new iterator

my_for("abc", print)
print(list(my_zip([1, 2, 3], "ab")))          # [(1, 'a'), (2, 'b')]
print(list(my_enumerate(["x", "y", "z"], 1))) # [(1,'x'), (2,'y'), (3,'z')]
```

---

## Exercise 3 — Exhaustion & Re-iteration

```python
import itertools


class AlreadyExhaustedError(Exception): pass


class ReusableIterator:
    """Stores the SOURCE — yields a fresh iterator on every __iter__ call."""
    def __init__(self, source):
        self._source = source

    def __iter__(self):
        return iter(self._source)


class SinglePassIterator:
    """Raises on second iteration rather than silently returning nothing."""
    def __init__(self, source):
        self._iter = iter(source)
        self._used = False

    def __iter__(self):
        if self._used:
            raise AlreadyExhaustedError("Already consumed")
        self._used = True
        return self._iter

    def __next__(self):
        return next(self._iter)


# Exhaustion demo
lst = [1, 2, 3, 4, 5]
it = iter(lst)
print(next(it), next(it))   # 1 2
print(list(it))             # [3, 4, 5] — half consumed
print(list(it))             # []        — exhausted
print(list(lst))            # [1, 2, 3, 4, 5] — list always resets


# Classic double-iteration bug
def count_and_list(iterable):
    items = list(iterable)
    count = len(items)
    second_pass = list(iterable)   # empty if iterable is an exhausted iterator
    return count, second_pass

it = iter([10, 20, 30])
print(count_and_list(it))           # (3, []) — BUG: second pass empty
print(count_and_list([10, 20, 30])) # (3, [10, 20, 30]) — correct: pass the list


# itertools.tee — fork one iterator into two independent copies
it = iter(range(5))
a, b = itertools.tee(it, 2)
print(list(a))   # [0, 1, 2, 3, 4]
print(list(b))   # [0, 1, 2, 3, 4] — independent
```

---

## Exercise 4 — `__iter__` and `__next__`

```python
import itertools


class DBCursor:
    def __init__(self, rows: list):
        self._rows = rows
        self._pos = 0

    def __iter__(self):
        return self   # iterator protocol: must return self

    def __next__(self):
        if self._pos >= len(self._rows):
            raise StopIteration
        row = self._rows[self._pos]
        self._pos += 1
        return row

    def peek(self):
        """Return next item without advancing position."""
        if self._pos >= len(self._rows):
            raise StopIteration
        return self._rows[self._pos]

    def reset(self):
        self._pos = 0


class LimitedCursor(DBCursor):
    def __init__(self, rows: list, limit: int):
        super().__init__(rows)
        self._limit = limit
        self._yielded = 0

    def __next__(self):
        if self._yielded >= self._limit:
            raise StopIteration
        row = super().__next__()
        self._yielded += 1
        return row


class InfiniteCounter:
    def __init__(self, start=0, step=1):
        self._current = start
        self._step = step

    def __iter__(self): return self

    def __next__(self):
        val = self._current
        self._current += self._step
        return val


rows = [{"id": i} for i in range(10)]
cur = DBCursor(rows)

print(iter(cur) is cur)    # True — iterator contract
print(cur.peek())          # {"id": 0} — no advance
print(next(cur))           # {"id": 0} — advances
cur.reset()
print(list(cur))           # all 10 rows

lim = LimitedCursor(rows, 3)
print(list(lim))           # first 3 rows only

# Safe usage of infinite iterator via islice
counter = InfiniteCounter(0, 2)
print(list(itertools.islice(counter, 5)))   # [0, 2, 4, 6, 8]
```

---

## Exercise 5 — Internal State & One-Pass Traversal

```python
import re


class Tokenizer:
    TOKEN_RE = re.compile(r'\d+\.\d+|\d+|[A-Za-z_]\w*|[^\s\w]')

    def __init__(self, source: str):
        self._matches = list(self.TOKEN_RE.finditer(source))
        self._pos = 0

    def __iter__(self): return self

    def __next__(self):
        if self._pos >= len(self._matches):
            raise StopIteration
        token = self._matches[self._pos].group()
        self._pos += 1
        return token


class ZipIterator:
    def __init__(self, iter_a, iter_b):
        self._a = iter(iter_a)
        self._b = iter(iter_b)

    def __iter__(self): return self

    def __next__(self):
        a = next(self._a)   # propagates StopIteration — stops when either exhausted
        b = next(self._b)
        return (a, b)


class ChainIterator:
    def __init__(self, *iterables):
        self._sources = [iter(i) for i in iterables]
        self._current_idx = 0

    def __iter__(self): return self

    def __next__(self):
        while self._current_idx < len(self._sources):
            try:
                return next(self._sources[self._current_idx])
            except StopIteration:
                self._current_idx += 1
        raise StopIteration


tok1 = Tokenizer("price = 3.14 * qty")
tok2 = Tokenizer("price = 3.14 * qty")
print(next(tok1))   # price
print(next(tok1))   # =
print(next(tok2))   # price — independent, unaffected by tok1

print(list(ZipIterator([1, 2, 3], "ab")))       # [(1,'a'), (2,'b')]
print(list(ChainIterator([1, 2], [3, 4], [5]))) # [1, 2, 3, 4, 5]

# All satisfy iter(it) is it
for it_obj in [tok1, ZipIterator([], []), ChainIterator([])]:
    print(iter(it_obj) is it_obj)   # True
```

---

## Exercise 6 — `StopIteration` & Exhaustion Patterns

```python
import itertools


def safe_next(iterator, default=None):
    try:
        return next(iterator)
    except StopIteration:
        return default

it = iter([1, 2])
print(safe_next(it))      # 1
print(safe_next(it))      # 2
print(safe_next(it, -1))  # -1
print(next(it, -1))       # -1 — built-in equivalent


class SentinelIterator:
    def __init__(self, source, sentinel):
        self._it = iter(source)
        self._sentinel = sentinel

    def __iter__(self): return self

    def __next__(self):
        val = next(self._it)
        if val == self._sentinel:
            raise StopIteration
        return val

print(list(SentinelIterator(["hello", "world", "STOP", "ignored"], "STOP")))
# ['hello', 'world']


# PEP 479: StopIteration inside a generator becomes RuntimeError
def broken_gen():
    it = iter([1, 2])
    while True:
        yield next(it)   # next() raises StopIteration — PEP 479 converts to RuntimeError

try:
    list(broken_gen())
except RuntimeError as e:
    print(f"PEP 479: {e}")

# Fix: catch StopIteration explicitly
def fixed_gen():
    it = iter([1, 2])
    while True:
        try:
            yield next(it)
        except StopIteration:
            return

print(list(fixed_gen()))   # [1, 2]


class BufferedIterator:
    def __init__(self, source, buffer_size: int):
        self._it = iter(source)
        self._buf = []
        self._buf_size = buffer_size
        self._fill()

    def _fill(self):
        while len(self._buf) < self._buf_size:
            try:
                self._buf.append(next(self._it))
            except StopIteration:
                break

    def has_next(self) -> bool:
        return bool(self._buf)

    def __iter__(self): return self

    def __next__(self):
        if not self._buf:
            raise StopIteration
        val = self._buf.pop(0)
        self._fill()
        return val

print(list(BufferedIterator(range(7), buffer_size=3)))   # [0, 1, 2, 3, 4, 5, 6]


def first_n(iterable, n):
    it = iter(iterable)
    for _ in range(n):
        try:
            yield next(it)
        except StopIteration:
            return

def skip_n(iterable, n):
    it = iter(iterable)
    for _ in range(n):
        try:
            next(it)
        except StopIteration:
            return
    yield from it

print(list(first_n(range(10), 4)))   # [0, 1, 2, 3]
print(list(skip_n(range(10), 6)))    # [6, 7, 8, 9]
```

---

## Exercise 7 — Generator Functions & `yield`

```python
import inspect


def read_transactions(simulated_csv: list):
    """Generator: one parsed dict per row — never loads full file."""
    for line in simulated_csv:
        if line.startswith("#"):
            continue
        id_, amount, currency = line.split(",")
        yield {"id": id_.strip(), "amount": float(amount), "currency": currency.strip()}


def running_total(amounts):
    total = 0.0
    for amount in amounts:
        total += amount
        yield total


def sliding_window(iterable, n: int):
    it = iter(iterable)
    window = []
    try:
        for _ in range(n):
            window.append(next(it))
    except StopIteration:
        return
    yield tuple(window)
    for item in it:
        window.pop(0)
        window.append(item)
        yield tuple(window)


def interleave(*iterables):
    iterators = [iter(i) for i in iterables]
    active = list(range(len(iterators)))
    while active:
        still_active = []
        for idx in active:
            try:
                yield next(iterators[idx])
                still_active.append(idx)
            except StopIteration:
                pass   # this source exhausted — drop it
        active = still_active


# Body does NOT execute until first next()
def lazy_gen():
    print("generator started")
    yield 1
    yield 2

gen = lazy_gen()
print("generator created — nothing printed yet")
print(next(gen))   # "generator started" prints NOW

# Internal state inspection
gen2 = lazy_gen()
next(gen2)
print(gen2.gi_frame)     # <frame object> — suspended
print(gen2.gi_running)   # False
try:
    next(gen2); next(gen2)
except StopIteration:
    pass
print(gen2.gi_frame)     # None — exhausted

# Generator is both iterator and iterable
gen3 = lazy_gen()
print(iter(gen3) is gen3)   # True

# Usage
csv_data = ["t1, 100.0, USD", "# comment", "t2, 200.5, EUR"]
for tx in read_transactions(csv_data):
    print(tx)

print(list(running_total([10, 20, 30, 40])))     # [10, 30, 60, 100]
print(list(sliding_window([1, 2, 3, 4, 5], 3)))  # [(1,2,3),(2,3,4),(3,4,5)]
print(list(interleave([1, 2, 3], [10, 20], [100])))  # [1,10,100,2,20,3]
```

---

## Exercise 8 — Generator Expressions

```python
import sys


lines = ["  ERROR|svc|disk full  ", "  INFO|svc|ok  ", "  ERROR|app|oom  "]
tokens = ["42", "hello", "7", "world", "99"]

# Equivalent generator expressions
gen_squares   = (x**2 for x in range(1_000_000))
gen_errors    = (line.strip() for line in lines if line.strip().startswith("ERROR"))
gen_integers  = (int(x) for x in tokens if x.isdigit())

# Memory comparison
list_size = sys.getsizeof([x**2 for x in range(1_000_000)])
gen_size  = sys.getsizeof(x**2 for x in range(1_000_000))
print(f"List: {list_size:,} bytes | Generator: {gen_size} bytes")
# Typically ~8MB vs ~200 bytes

# Lazy chain — nothing processes until consumed
raw_lines = ["  ERROR|svc|disk full  ", "  INFO|svc|ok  ", "  ERROR|app|oom  "]
pipeline_lines    = (line.strip() for line in raw_lines)
pipeline_errors   = (l for l in pipeline_lines if "ERROR" in l)
pipeline_messages = (l.split("|")[2] for l in pipeline_errors)

print("pipeline built — nothing processed yet")
print(next(pipeline_messages))   # "disk full" — only NOW does any work happen


# short-circuit: any() stops at first True
print(any(x > 5 for x in range(1_000_000)))   # stops at 6, doesn't process rest


def pipeline(seed, *fns):
    """Chain transforms lazily without intermediate lists."""
    result = seed
    for fn in fns:
        result = (fn(item) for item in result)
    return result

processed = pipeline(
    ["  hello  ", "  WORLD  ", "  foo  "],
    str.strip,
    str.lower,
    lambda s: s + "!"
)
print(list(processed))   # ['hello!', 'world!', 'foo!']


# Scope trap in loops
# List comprehension has its OWN scope in Python 3 — i is local
list_fns = [lambda i=i: i for i in range(5)]   # needs default arg capture
print([f() for f in list_fns])   # [0, 1, 2, 3, 4]

# Generator expression captures enclosing i BY REFERENCE — same trap as closures
i = 99
gen_trap = (i for _ in range(3))   # captures the NAME 'i', not its value
# Don't reassign i here — demonstrate the reference capture
print(list(gen_trap))   # [99, 99, 99] — all see the same i
```

---

## Exercise 9 — `yield from`

```python
def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)   # recursive delegation
        else:
            yield item

print(list(flatten([1, [2, [3, [4]], 5], 6]]))   # [1, 2, 3, 4, 5, 6]


def walk_tree(node: dict):
    yield node["value"]
    for child in node.get("children", []):
        yield from walk_tree(child)   # depth-first via delegation

tree = {"value": 1, "children": [
    {"value": 2, "children": [
        {"value": 4, "children": []},
        {"value": 5, "children": []}
    ]},
    {"value": 3, "children": []}
]}
print(list(walk_tree(tree)))   # [1, 2, 4, 5, 3]


def chain_generators(*gens):
    for gen in gens:
        yield from gen

print(list(chain_generators([1, 2], [3, 4], [5])))   # [1, 2, 3, 4, 5]


# yield from captures sub-generator RETURN VALUE
def sub_producer():
    yield 1
    yield 2
    return "sub_done"   # carried in StopIteration.value

def delegating():
    result = yield from sub_producer()   # 'result' gets "sub_done"
    print(f"sub returned: {result}")
    yield 99

gen = delegating()
print(next(gen))   # 1
print(next(gen))   # 2
# sub returned: sub_done  ← printed inside delegating()
print(next(gen))   # 99


# Accessing StopIteration.value manually
def producer():
    yield 10
    yield 20
    return "final_value"

gen = producer()
try:
    while True:
        print(next(gen))
except StopIteration as e:
    print(f"Return value: {e.value}")   # "final_value"


# yield from transparently proxies send() — a manual loop cannot do this
def accumulator():
    """Receives values, delegates to producer, returns total."""
    total = 0
    # yield from passes every send() value directly into sub_producer
    # A manual 'for item in sub: yield item' loses the send() transparency
    result = yield from sub_producer()
    return result
```

---

## Exercise 10 — `send()`, `throw()`, `close()`

```python
import functools


def coroutine(fn):
    """Auto-prime decorator — advances to first yield automatically."""
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        gen = fn(*args, **kwargs)
        next(gen)   # prime
        return gen
    return wrapper


# --- send() ---

def running_average():
    """Must be primed manually: next(gen) before send(value)."""
    total = 0.0
    count = 0
    while True:
        value = yield (total / count if count else 0)
        total += value
        count += 1

avg = running_average()
next(avg)              # prime — advances to first yield
print(avg.send(10))    # 10.0
print(avg.send(20))    # 15.0
print(avg.send(30))    # 20.0


@coroutine
def echo_transformer(transform_fn):
    """Receives values via send(), yields transformed result."""
    result = None
    while True:
        value = yield result
        result = transform_fn(value)

upper   = echo_transformer(str.upper)
trimmer = echo_transformer(str.strip)

# Chain: send to upper, then send its output to trimmer
result_upper = upper.send("  hello world  ")
print(repr(result_upper))               # '  HELLO WORLD  '
result_final = trimmer.send(result_upper)
print(repr(result_final))               # 'HELLO WORLD'


# --- throw() ---

def resilient_counter():
    """Resets to 0 when ValueError is thrown; exits cleanly on GeneratorExit."""
    count = 0
    while True:
        try:
            yield count
            count += 1
        except ValueError:
            print(f"Reset! (was at {count})")
            count = 0

rc = resilient_counter()
print(next(rc))   # 0
print(next(rc))   # 1
print(next(rc))   # 2
rc.throw(ValueError, "reset")  # Reset! (was at 3)
print(next(rc))   # 0
print(next(rc))   # 1
# throw() delivers the exception AT THE POINT OF SUSPENSION (inside the yield expression)
# The try/except inside the generator body handles it normally


# --- close() ---

def resource_generator(resource_name: str):
    print(f"[OPEN] {resource_name}")
    try:
        for i in range(100):
            yield f"{resource_name}:item_{i}"
    finally:
        # Runs whether exhausted normally OR closed mid-iteration via .close()
        print(f"[CLOSE] {resource_name}")

gen = resource_generator("db_conn")
print(next(gen))   # [OPEN] db_conn  →  db_conn:item_0
print(next(gen))   # db_conn:item_1
gen.close()        # [CLOSE] db_conn — guaranteed cleanup


# GeneratorExit — close() throws it at the suspension point
def gen_with_exit():
    try:
        while True:
            yield
    except GeneratorExit:
        print("Cleaning up on close()")
        return   # correct — return or fall off; re-raising is also acceptable
        # Wrong: yield after catching GeneratorExit => RuntimeError

g = gen_with_exit()
next(g)
g.close()   # "Cleaning up on close()"


# --- Controlled pipeline combining all three ---
@coroutine
def controlled_pipeline(source):
    """
    send(batch_size) → yields next batch_size items from source.
    throw(RetrySignal) → re-yields the last batch.
    close() → triggers cleanup via GeneratorExit / finally.
    """
    it = iter(source)
    last_batch = []
    try:
        while True:
            try:
                batch_size = yield last_batch
                last_batch = []
                for _ in range(batch_size or 1):
                    try:
                        last_batch.append(next(it))
                    except StopIteration:
                        break
                if not last_batch:
                    return
            except ValueError:
                print("[RETRY] re-sending last batch")
                # last_batch is unchanged — will be re-yielded next iteration
    finally:
        print("[PIPELINE] closed cleanly")


data = list(range(20))
pipe = controlled_pipeline(data)

print(pipe.send(3))    # [0, 1, 2]
print(pipe.send(4))    # [3, 4, 5, 6]
pipe.throw(ValueError) # [RETRY]
print(pipe.send(2))    # [3, 4] — re-sent last batch... then fetches 2 more
pipe.close()           # [PIPELINE] closed cleanly
```

---

## Bonus — Full Lazy Streaming Pipeline

```python
import functools
import time
from datetime import datetime, UTC


# --- Infrastructure ---

def coroutine(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        gen = fn(*args, **kwargs)
        next(gen)
        return gen
    return wrapper


class DataSource:
    """Reusable iterable — fresh iterator on every __iter__ call."""
    def __init__(self, records: list):
        self._records = records

    def __iter__(self):
        return iter(self._records)


class TransformIterator:
    """Lazy iterator — applies transform_fn to each element on demand."""
    def __init__(self, source, transform_fn):
        self._it = iter(source)
        self._fn = transform_fn

    def __iter__(self): return self

    def __next__(self):
        return self._fn(next(self._it))


# --- Pipeline stages ---

def normalize(record: dict) -> dict:
    return {k: (v.strip() if isinstance(v, str) else v) for k, v in record.items()}

def require_fields(record: dict):
    missing = [f for f in ("id", "amount") if f not in record]
    return (False, f"missing: {missing}") if missing else (True, "")

def positive_amounts(record: dict):
    if record.get("amount", 0) <= 0:
        return (False, f"non-positive: {record.get('amount')}")
    return (True, "")


def validate_stream(iterable, *validators):
    """Generator: yields (record, [error_messages]) for every record."""
    for record in iterable:
        errors = [msg for _, msg in [v(record) for v in validators]
                  if _ is False]
        # Rebuild with correct unpacking:
        errors = []
        for validator in validators:
            ok, msg = validator(record)
            if not ok:
                errors.append(msg)
        yield record, errors


def batch(iterable, size: int):
    """Generator: groups items into lists of `size`."""
    buf = []
    for item in iterable:
        buf.append(item)
        if len(buf) == size:
            yield buf
            buf = []
    if buf:
        yield buf


def throttle(iterable, max_per_second: int):
    """Generator: enforces rate limit between yields."""
    interval = 1.0 / max_per_second
    for item in iterable:
        yield item
        time.sleep(interval)


def enrich_with_metadata(record: dict) -> dict:
    return {**record, "processed_at": datetime.now(UTC).isoformat()}


@coroutine
def sink(transform_fn):
    """
    Coroutine sink: receives records via send(), transforms each.
    Send None as sentinel to terminate and retrieve results via StopIteration.value.
    """
    results = []
    while True:
        record = yield len(results)   # yield current count as status feedback
        if record is None:            # sentinel: caller signals end of stream
            return results
        results.append(transform_fn(record))


# --- Assemble and run ---

raw_records = [
    {"id": "t1", "amount": 100.0,  "currency": "USD "},
    {"id": "t2", "amount": -50.0,  "currency": "EUR"},   # invalid
    {"id": "t3", "amount": 200.0,  "currency": "GBP"},
    {"id": "t4", "amount": 0.0,    "currency": "USD"},   # invalid
    {"id": "t5", "amount": 75.0,   "currency": "JPY"},
]

source = DataSource(raw_records)

valid_records = (
    rec
    for rec, errs in validate_stream(
        TransformIterator(source, normalize),
        require_fields,
        positive_amounts,
    )
    if not errs
)

# batch(throttle(...), size=2) — whole chain is lazy until iterated
pipeline = batch(
    throttle(valid_records, max_per_second=1000),
    size=2
)

collector = sink(enrich_with_metadata)

for batch_group in pipeline:
    for record in batch_group:
        collector.send(record)

# Terminate with sentinel, capture results from StopIteration.value
try:
    collector.send(None)
except StopIteration as e:
    final_results = e.value
    print(f"Processed {len(final_results)} valid records:")
    for r in final_results:
        print(f"  {r['id']} | {r['amount']} | enriched={bool(r.get('processed_at'))}")

# Output:
# Processed 3 valid records:
#   t1 | 100.0 | enriched=True
#   t3 | 200.0 | enriched=True
#   t5 | 75.0  | enriched=True
```
