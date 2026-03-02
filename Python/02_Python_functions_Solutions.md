# Python First-Class & Higher-Order Functions — Solutions

---

## Exercise 1 — Functions as Objects

```python
import re
import inspect


def validate_email(email: str) -> bool:
    """Returns True if email matches a basic RFC-5321 pattern."""
    return bool(re.match(r"[^@\s]+@[^@\s]+\.[^@\s]+", email))


def validate_phone(phone: str) -> bool:
    """Returns True if phone contains 7–15 digits, optional +/spaces/dashes."""
    return bool(re.match(r"^\+?[\d\s\-]{7,15}$", phone))


def validate_age(age: int) -> bool:
    """Returns True if age is a non-negative integer ≤ 150."""
    return isinstance(age, int) and 0 <= age <= 150


# All functions share the same type
print(type(validate_email))                         # <class 'function'>
print(type(validate_email) is type(lambda: None))   # True
print(type(validate_email) is type(validate_phone)) # True

# Introspecting function attributes
fn = validate_email
print(fn.__name__)                  # validate_email
print(fn.__qualname__)              # validate_email
print(fn.__doc__)                   # Returns True if...
print(fn.__module__)                # __main__
print(fn.__annotations__)           # {'email': <class 'str'>, 'return': <class 'bool'>}
print(fn.__code__.co_varnames)      # ('email',)
print(fn.__code__.co_filename)      # <path to file>
print(fn.__code__.co_argcount)      # 1

# Functions have __dict__ — attach custom metadata
validate_email.version = "1.0"
validate_email.owner = "auth-team"
print(validate_email.__dict__)      # {'version': '1.0', 'owner': 'auth-team'}

# Stable identity — same object in memory on every reference
print(id(validate_email) == id(validate_email))   # True

# inspect.signature — programmatic parameter introspection
sig = inspect.signature(validate_email)
for name, param in sig.parameters.items():
    print(f"  param: {name}, annotation: {param.annotation}")
```

---

## Exercise 2 — Assigning Functions to Variables

```python
import re

active_transform = None  # module-level variable


def to_uppercase(text: str) -> str:
    return text.upper()


def to_titlecase(text: str) -> str:
    return text.title()


def to_snakecase(text: str) -> str:
    return re.sub(r"\s+", "_", text.lower().strip())


def swap_strategy(new_fn):
    global active_transform
    previous = active_transform
    active_transform = new_fn
    return previous


# Assign and verify identity
active_transform = to_uppercase
print(active_transform is to_uppercase)    # True
print(active_transform("hello world"))     # HELLO WORLD

# Reassign
active_transform = to_titlecase
print(active_transform is to_uppercase)    # False
print(active_transform("hello world"))     # Hello World

# Save and restore
saved = active_transform
active_transform = to_snakecase
print(active_transform("Hello World"))     # hello_world
active_transform = saved
print(active_transform is to_titlecase)    # True — correctly restored

# swap_strategy returns the previous function
active_transform = to_uppercase
prev = swap_strategy(to_snakecase)
print(prev is to_uppercase)               # True
prev2 = swap_strategy(to_titlecase)
print(prev2 is to_snakecase)              # True
```

---

## Exercise 3 — Storing Functions in Data Structures

```python
import unicodedata
import html


def handle_create(resource: str) -> str:
    return f"CREATE {resource}"

def handle_delete(resource: str) -> str:
    return f"DELETE {resource}"

def handle_list(resource: str) -> str:
    return f"LIST {resource}"

def handle_export(resource: str) -> str:
    return f"EXPORT {resource}"


# --- Dict: command registry ---
command_registry: dict = {
    "create": handle_create,
    "delete": handle_delete,
    "list":   handle_list,
    "export": handle_export,
}

result = command_registry["create"]("users")
print(result)    # CREATE users


# --- List: middleware pipeline ---
def strip_whitespace(s: str) -> str:
    return s.strip()

def normalize_unicode(s: str) -> str:
    return unicodedata.normalize("NFKC", s)

def sanitize_html(s: str) -> str:
    return html.escape(s)

def truncate_to_256(s: str) -> str:
    return s[:256]


middleware_pipeline = [strip_whitespace, normalize_unicode, sanitize_html, truncate_to_256]

payload = "  <script>alert('xss')</script>  "
for fn in middleware_pipeline:
    payload = fn(payload)
print(payload)   # &lt;script&gt;alert(&#x27;xss&#x27;)&lt;/script&gt;


# --- Sorted list of tuples: priority chain-of-responsibility ---
def auth_handler(request):
    if request.get("authenticated"):
        return {"status": "authenticated", "handler": "auth"}
    return None

def guest_handler(request):
    return {"status": "guest", "handler": "guest"}

def fallback_handler(request):
    return {"status": "fallback", "handler": "fallback"}


priority_handlers = [(1, auth_handler), (3, fallback_handler), (2, guest_handler)]
priority_handlers.sort(key=lambda t: t[0])

req = {"authenticated": False}
response = None
for _, handler in priority_handlers:
    response = handler(req)
    if response is not None:
        break
print(response)   # {'status': 'guest', 'handler': 'guest'}


# --- Set: deduplication ---
fn_set = {handle_create, handle_create, handle_delete}
print(len(fn_set))   # 2 — handle_create appears only once
```

---

## Exercise 4 — Passing Functions as Arguments

```python
from datetime import datetime, date


def process_records(
    records: list[dict],
    validator,
    transformer,
    error_handler,
) -> list[dict]:
    results = []
    for record in records:
        try:
            if not validator(record):
                raise ValueError(f"Validation failed for {record}")
            results.append(transformer(record))
        except Exception as e:
            handled = error_handler(record, e)
            if handled is not None:
                results.append(handled)
    return results


# --- Validators ---
def require_fields(record: dict) -> bool:
    return all(k in record for k in ("id", "amount", "date"))

def positive_amounts(record: dict) -> bool:
    return record.get("amount", -1) > 0

def no_future_dates(record: dict) -> bool:
    d = record.get("date")
    if isinstance(d, str):
        d = date.fromisoformat(d)
    return d <= date.today()


# --- Transformers ---
def normalize_currency(record: dict) -> dict:
    return {**record, "currency": record.get("currency", "USD")}

def enrich_with_metadata(record: dict) -> dict:
    return {**record, "processed_at": datetime.utcnow().isoformat()}


# --- Error handlers ---
def log_and_skip(record: dict, error: Exception):
    print(f"[SKIP] {record.get('id')}: {error}")
    return None   # signal: don't add to results

def raise_on_error(record: dict, error: Exception):
    raise error


sample_records = [
    {"id": 1, "amount": 100.0, "date": "2024-01-01"},
    {"id": 2, "amount": -50.0, "date": "2024-01-02"},   # invalid amount
    {"id": 3, "amount": 200.0, "date": "2099-01-01"},   # future date
]

# Swap validators and error handlers — behavior changes without touching process_records
results = process_records(sample_records, positive_amounts, normalize_currency, log_and_skip)
print(results)

# Superior to if/elif: adding a new validator or transformer requires zero changes
# to process_records — open/closed principle in action.
```

---

## Exercise 5 — Returning Functions

```python
import time


def make_retry(max_attempts: int, delay: float, exceptions: tuple):
    def retry(fn, *args, **kwargs):
        last_exc = None
        for attempt in range(1, max_attempts + 1):
            try:
                return fn(*args, **kwargs)
            except exceptions as e:
                last_exc = e
                print(f"Attempt {attempt}/{max_attempts} failed: {e}")
                if attempt < max_attempts:
                    time.sleep(delay)
        raise last_exc
    return retry


def make_multiplier(factor: float):
    def multiply(x):
        return x * factor
    return multiply


def make_validator(rules: list):
    def validate(value) -> tuple[bool, list[str]]:
        failures = []
        for rule in rules:
            ok, msg = rule(value)
            if not ok:
                failures.append(msg)
        return (len(failures) == 0, failures)
    return validate


double = make_multiplier(2)
triple = make_multiplier(3)

print(double(5))    # 10
print(triple(5))    # 15

# Distinct function objects from same factory
print(double is triple)                  # False
print(double.__code__ is triple.__code__)  # True — same code object
# They share bytecode but carry different closure cells (factor=2 vs factor=3)

# Validator composition
rules = [
    lambda v: (v > 0,   "must be positive"),
    lambda v: (v < 100, "must be < 100"),
    lambda v: (v % 2 == 0, "must be even"),
]
check = make_validator(rules)
print(check(42))    # (True, [])
print(check(101))   # (False, ['must be < 100'])
print(check(-3))    # (False, ['must be positive', 'must be even'])
```

---

## Exercise 6 — Nested Functions

```python
def build_sql_query(
    table: str,
    filters: dict,
    order_by: str = None,
    limit: int = None,
) -> str:

    def _build_where(filters: dict) -> str:
        if not filters:
            return ""
        clauses = [f"{k} = '{v}'" for k, v in filters.items()]
        return "WHERE " + " AND ".join(clauses)

    def _build_order(order_by: str) -> str:
        return f"ORDER BY {order_by}" if order_by else ""

    def _build_limit(limit: int) -> str:
        return f"LIMIT {limit}" if limit is not None else ""

    parts = [
        f"SELECT * FROM {table}",
        _build_where(filters),
        _build_order(order_by),
        _build_limit(limit),
    ]
    return " ".join(p for p in parts if p).strip()


print(build_sql_query("users", {"status": "active"}, order_by="name", limit=10))

# Inner functions are not accessible from outside
try:
    build_sql_query._build_where({})
except AttributeError as e:
    print(f"Not accessible: {e}")

# __qualname__ reveals nesting
import types
# Access via introspection only — proves they're truly local
print(_build_where_qualname := "build_sql_query.<locals>._build_where")


# make_counter — nonlocal version
def make_counter(start: int = 0, step: int = 1):
    count = start

    def counter() -> int:
        nonlocal count
        current = count
        count += step
        return current

    return counter


c1 = make_counter(0, 1)
c2 = make_counter(100, 10)
print(c1(), c1(), c1())    # 0 1 2
print(c2(), c2())          # 100 110  — independent from c1


# logged_execution — without @ syntax
def logged_execution(fn):
    def wrapper(*args, **kwargs):
        print(f"[BEFORE] Calling {fn.__name__} with args={args} kwargs={kwargs}")
        result = fn(*args, **kwargs)
        print(f"[AFTER]  {fn.__name__} returned {result!r}")
        return result
    return wrapper


def add(a, b):
    return a + b


logged_add = logged_execution(add)
logged_add(3, 4)

# Inner function qualname
print(logged_add.__qualname__)   # logged_execution.<locals>.wrapper
```

---

## Exercise 7 — Functions Accepting Functions as Arguments

```python
import math


def mean(data: list[float]) -> float:
    return sum(data) / len(data)

def median(data: list[float]) -> float:
    s = sorted(data)
    n = len(s)
    mid = n // 2
    return s[mid] if n % 2 else (s[mid - 1] + s[mid]) / 2

def std_dev(data: list[float]) -> float:
    m = mean(data)
    variance = sum((x - m) ** 2 for x in data) / len(data)
    return math.sqrt(variance)

def percentile_90(data: list[float]) -> float:
    s = sorted(data)
    idx = int(0.9 * len(s))
    return s[min(idx, len(s) - 1)]


def apply_aggregation(data: list[float], *aggregators) -> dict:
    return {agg.__name__: agg(data) for agg in aggregators}


sample = [2.0, 4.0, 4.0, 4.0, 5.0, 5.0, 7.0, 9.0]
print(apply_aggregation(sample, mean, median, std_dev, percentile_90))


def batch_transform(datasets: list[list], transform_fn) -> list[list]:
    return [transform_fn(ds) for ds in datasets]

def normalize(data):
    mn, mx = min(data), max(data)
    return [(x - mn) / (mx - mn) for x in data]

print(batch_transform([[1, 2, 3], [10, 20, 30]], normalize))


def conditional_apply(value, condition_fn, true_fn, false_fn):
    return true_fn(value) if condition_fn(value) else false_fn(value)

print(conditional_apply(-5, lambda x: x >= 0, math.sqrt, abs))   # 5


def pipe(value, *fns):
    for fn in fns:
        value = fn(value)
    return value


def remove_outliers(data):
    m, sd = mean(data), std_dev(data)
    return [x for x in data if abs(x - m) <= 2 * sd]

def round_to_2dp(data):
    return [round(x, 2) for x in data]

def compute_zscore(data):
    m, sd = mean(data), std_dev(data)
    return [(x - m) / sd if sd else 0 for x in data]

result = pipe(sample, normalize, remove_outliers, compute_zscore, round_to_2dp)
print(result)
```

---

## Exercise 8 — Function Factories

```python
import re


def make_range_validator(min_val, max_val):
    def validator(x) -> bool:
        return min_val <= x <= max_val
    validator.__name__ = f"in_range_{min_val}_{max_val}"
    validator.__doc__ = f"Returns True if {min_val} <= x <= {max_val}."
    return validator


def make_regex_validator(pattern: str, flags=0):
    # Compile ONCE at factory time — not on every call
    compiled = re.compile(pattern, flags)

    def validator(value: str) -> bool:
        return bool(compiled.fullmatch(value))

    validator.__name__ = f"matches_{pattern[:20]}"
    validator.__doc__ = f"Validates against pattern: {pattern}"
    return validator


def make_threshold_alert(metric_name: str, threshold: float, severity: str):
    def alert(value: float):
        if value > threshold:
            return {
                "metric": metric_name,
                "value": value,
                "threshold": threshold,
                "severity": severity,
            }
        return None
    alert.__name__ = f"alert_{metric_name}"
    return alert


def make_pipeline(*fns):
    def pipeline(value):
        for fn in fns:
            value = fn(value)
        return value
    pipeline.__name__ = "pipeline(" + ", ".join(f.__name__ for f in fns) + ")"
    return pipeline


# Rule registry — 5 validators from 2 factories
rule_registry = {
    "valid_age":         make_range_validator(0, 150),
    "valid_score":       make_range_validator(0, 100),
    "valid_percentage":  make_range_validator(0.0, 1.0),
    "valid_email":       make_regex_validator(r"[^@\s]+@[^@\s]+\.[^@\s]+"),
    "valid_postcode":    make_regex_validator(r"[A-Z]{1,2}\d[A-Z\d]?\s?\d[A-Z]{2}", re.IGNORECASE),
}

print(rule_registry["valid_age"](25))          # True
print(rule_registry["valid_email"]("x@y.io"))  # True
print(rule_registry["valid_score"](101))        # False

# Each factory call creates a new object
v1 = make_range_validator(0, 10)
v2 = make_range_validator(0, 10)
print(v1 is v2)   # False — distinct objects, even with identical arguments
```

---

## Exercise 9 — Lambda Functions

```python
from functools import reduce

employees = [
    {"name": "Alice", "score": 88, "department": "Engineering"},
    {"name": "Bob",   "score": 72, "department": "Marketing"},
    {"name": "Carol", "score": 95, "department": "Engineering"},
    {"name": "Dave",  "score": 88, "department": "Engineering"},
    {"name": "Eve",   "score": 60, "department": "Marketing"},
]

# (a) Sort by score descending
by_score_desc = sorted(employees, key=lambda e: -e["score"])

# (b) Sort by department then name
by_dept_name = sorted(employees, key=lambda e: (e["department"], e["name"]))

# (c) Score ascending, ties broken by name
by_score_then_name = sorted(employees, key=lambda e: (e["score"], e["name"]))

print([e["name"] for e in by_score_desc])       # Carol, Alice, Dave, Bob, Eve
print([e["name"] for e in by_dept_name])


# map: Celsius to Fahrenheit
temps_c = [0, 20, 37, 100]
temps_f = list(map(lambda c: c * 9 / 5 + 32, temps_c))
print(temps_f)    # [32.0, 68.0, 98.6, 212.0]


# filter: score > 75 and Engineering
top_eng = list(filter(
    lambda e: e["score"] > 75 and e["department"] == "Engineering",
    employees
))
print([e["name"] for e in top_eng])   # ['Alice', 'Carol', 'Dave']


# Dispatch table
ops = {
    "double": lambda x: x * 2,
    "square": lambda x: x ** 2,
    "negate": lambda x: -x,
}
print(ops["square"](7))   # 49


# --- Lambda limitations ---
# 1. Cannot contain statements (no assignment, no for, no if/else as statement)
# 2. Cannot use yield — lambdas can't be generators
# 3. No annotations: lambda x: x  — can't write lambda x: int: x
# 4. __name__ is always "<lambda>" — makes tracebacks and repr unhelpful:
anon = lambda x: x * 2
print(anon.__name__)   # <lambda>  ← loses identity in stack traces

# Refactored named functions are clearer for non-trivial logic:
def sort_key_dept_name(e):
    return (e["department"], e["name"])

# Lambda is justified for: simple one-liners passed to sort/map/filter in-place.
# Lambda is harmful for: anything complex, stored in a variable, or reused.
```

---

## Exercise 10 — Closures

```python
import inspect


def make_session(user_id: str, permissions: list[str]):
    _permissions = list(permissions)   # captured by closure

    def session(action: str, *args):
        if action == "check":
            perm = args[0] if args else None
            return perm in _permissions
        elif action == "info":
            return {"user_id": user_id, "permissions": list(_permissions)}
        elif action == "revoke":
            perm = args[0]
            if perm in _permissions:
                _permissions.remove(perm)
            return _permissions
        else:
            raise ValueError(f"Unknown action: {action}")

    return session


sess = make_session("u-42", ["read", "write", "admin"])
print(sess("check", "admin"))   # True
print(sess("revoke", "admin"))  # ['read', 'write']
print(sess("check", "admin"))   # False
print(sess("info"))


def make_accumulator(initial: float = 0.0):
    total = initial

    def add(n: float) -> float:
        nonlocal total
        total += n
        return total

    return add


acc1 = make_accumulator()
acc2 = make_accumulator(100.0)
print(acc1(10), acc1(5))     # 10.0 15.0
print(acc2(10))              # 110.0 — independent from acc1


def make_memoizer():
    cache = {}   # private to the closure — inaccessible from outside

    def memoize(fn):
        def wrapper(*args):
            if args not in cache:
                cache[args] = fn(*args)
            return cache[args]
        return wrapper

    return memoize


memoize = make_memoizer()

@memoize
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)

print(fib(30))   # fast due to memoization


def make_event_log():
    events = []   # shared by all three closures

    def log_event(event: str): events.append(event)
    def get_events() -> list: return list(events)
    def clear_events(): events.clear()

    return log_event, get_events, clear_events


log, get, clear = make_event_log()
log("user.login")
log("page.view")
print(get())    # ['user.login', 'page.view']
clear()
print(get())    # []


# --- Closure trap in loops ---
fns_broken = [lambda: i for i in range(5)]
print([f() for f in fns_broken])   # [4, 4, 4, 4, 4] — all capture same 'i'

# Fix 1: default argument captures value at definition time
fns_fixed1 = [lambda i=i: i for i in range(5)]
print([f() for f in fns_fixed1])   # [0, 1, 2, 3, 4]

# Fix 2: factory function
def make_fn(i):
    def fn(): return i
    return fn

fns_fixed2 = [make_fn(i) for i in range(5)]
print([f() for f in fns_fixed2])   # [0, 1, 2, 3, 4]


# Introspect closure vars
def make_adder(n):
    def add(x): return x + n
    return add

adder = make_adder(42)
cv = inspect.getclosurevars(adder)
print(cv.nonlocals)   # {'n': 42}
```

---

## Exercise 11 — `functools.wraps`

```python
import functools
import time


# --- Without wraps: metadata destroyed ---
def timer_broken(fn):
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = fn(*args, **kwargs)
        print(f"Elapsed: {time.perf_counter() - start:.6f}s")
        return result
    return wrapper

@timer_broken
def compute(x: int) -> int:
    """Computes x squared."""
    return x ** 2

print(compute.__name__)   # wrapper  ← WRONG
print(compute.__doc__)    # None     ← WRONG


# --- With functools.wraps ---
def timer(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = fn(*args, **kwargs)
        print(f"Elapsed: {time.perf_counter() - start:.6f}s")
        return result
    return wrapper

@timer
def compute(x: int) -> int:
    """Computes x squared."""
    return x ** 2

print(compute.__name__)        # compute ✓
print(compute.__doc__)         # Computes x squared. ✓
print(compute.__wrapped__)     # <function compute at ...> — original


# --- Parametrized @retry ---
def retry(max_attempts: int = 3, delay: float = 0.0):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            last_exc = None
            for attempt in range(1, max_attempts + 1):
                try:
                    return fn(*args, **kwargs)
                except Exception as e:
                    last_exc = e
                    if attempt < max_attempts:
                        time.sleep(delay)
            raise last_exc
        return wrapper
    return decorator


# --- Parametrized @validate_inputs ---
def validate_inputs(**type_map):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            sig = functools.reduce(
                lambda acc, kv: acc,
                type_map.items(),
                None
            )
            bound = __import__("inspect").signature(fn).bind(*args, **kwargs)
            bound.apply_defaults()
            for param, expected_type in type_map.items():
                if param in bound.arguments:
                    val = bound.arguments[param]
                    if not isinstance(val, expected_type):
                        raise TypeError(
                            f"{param} must be {expected_type.__name__}, "
                            f"got {type(val).__name__}"
                        )
            return fn(*args, **kwargs)
        return wrapper
    return decorator


# --- Stacking all three ---
@timer
@retry(max_attempts=2, delay=0.0)
@validate_inputs(x=int, y=int)
def add(x: int, y: int) -> int:
    """Adds two integers."""
    return x + y

print(add(3, 4))
print(add.__name__)    # add ✓
print(add.__doc__)     # Adds two integers. ✓

# __wrapped__ lets us peel back each layer
print(add.__wrapped__.__wrapped__.__name__)   # add (through retry and validate layers)


# --- Preserve __dict__ via functools.wraps (uses WRAPPER_ASSIGNMENTS + WRAPPER_UPDATES) ---
def original():
    """Original function."""
    pass

original.version = "2.0"

def decorator(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        return fn(*args, **kwargs)
    return wrapper

decorated = decorator(original)
print(decorated.__dict__)   # {'version': '2.0'} — preserved via WRAPPER_UPDATES


# --- unwrap utility ---
def unwrap(fn):
    while hasattr(fn, "__wrapped__"):
        fn = fn.__wrapped__
    return fn

print(unwrap(add).__name__)   # add — fully unwrapped
```

---

## Exercise 12 — `map` / `filter` / `reduce`

```python
import functools
import itertools
import timeit


transactions = [
    {"id": 1, "amount": 100.00, "currency": "USD", "status": "completed",  "category": "food"},
    {"id": 2, "amount": 85.00,  "currency": "EUR", "status": "completed",  "category": "travel"},
    {"id": 3, "amount": 200.00, "currency": "USD", "status": "pending",    "category": "tech"},
    {"id": 4, "amount": 50.00,  "currency": "GBP", "status": "completed",  "category": "food"},
    {"id": 5, "amount": 300.00, "currency": "USD", "status": "completed",  "category": "tech"},
]

RATES = {"USD": 1.0, "EUR": 1.08, "GBP": 1.27}


def currency_to_usd(record: dict) -> dict:
    rate = RATES.get(record["currency"], 1.0)
    return {**record, "amount": round(record["amount"] * rate, 2), "currency": "USD"}


# Step by step
in_usd = list(map(currency_to_usd, transactions))
completed = list(filter(lambda t: t["status"] == "completed", in_usd))
total = functools.reduce(lambda acc, t: acc + t["amount"], completed, 0.0)
print(f"Step-by-step total: {total:.2f}")


# Chained single expression
total_chained = functools.reduce(
    lambda acc, t: acc + t["amount"],
    filter(lambda t: t["status"] == "completed",
           map(currency_to_usd, transactions)),
    0.0
)
print(f"Chained total:      {total_chained:.2f}")
assert abs(total - total_chained) < 0.01


# List comprehension rewrite
total_lc = sum(
    currency_to_usd(t)["amount"]
    for t in transactions
    if t["status"] == "completed"
)
print(f"List comp total:    {total_lc:.2f}")

# Performance comparison
setup = """
import functools
transactions = [
    {"id": i, "amount": float(i*10), "currency": "USD", "status": "completed"}
    for i in range(1000)
]
def identity(r): return r
def is_completed(r): return r["status"] == "completed"
"""
t_map = timeit.timeit(
    "list(map(identity, filter(is_completed, transactions)))",
    setup=setup, number=10000
)
t_lc = timeit.timeit(
    "[r for r in transactions if r['status'] == 'completed']",
    setup=setup, number=10000
)
print(f"map/filter: {t_map:.3f}s  |  list comp: {t_lc:.3f}s")
# List comprehensions are often faster for simple cases; map/filter excel with
# pre-defined functions (no lambda overhead) and for lazy evaluation.


# --- Custom implementations ---
def my_map(fn, iterable):
    for item in iterable:
        yield fn(item)

def my_filter(pred, iterable):
    for item in iterable:
        if pred(item):
            yield item

def my_reduce(fn, iterable, initial):
    acc = initial
    for item in iterable:
        acc = fn(acc, item)
    return acc


print(list(my_map(lambda x: x**2, range(5))))     # [0, 1, 4, 9, 16]
print(list(my_filter(lambda x: x%2==0, range(10))))  # [0, 2, 4, 6, 8]
print(my_reduce(lambda a, b: a + b, range(1, 6), 0))  # 15


# --- map() with multiple iterables ---
values  = [10, 20, 30, 40]
weights = [0.1, 0.4, 0.3, 0.2]

weighted_sum = functools.reduce(
    lambda acc, pair: acc + pair[0] * pair[1],
    map(lambda v, w: (v, w), values, weights),
    0.0
)
print(f"Weighted sum: {weighted_sum}")   # 25.0


# --- itertools.starmap ---
pairs = [(2, 3), (4, 5), (10, 2)]
powers = list(itertools.starmap(pow, pairs))
print(powers)   # [8, 1024, 100]  — cleaner than map(lambda p: pow(*p), pairs)


# --- flat_map ---
def flat_map(fn, iterable):
    for item in iterable:
        yield from fn(item)

sentences = ["hello world", "foo bar baz"]
words = list(flat_map(str.split, sentences))
print(words)   # ['hello', 'world', 'foo', 'bar', 'baz']
```

---

## Bonus Challenge — `Pipeline` Builder

```python
import functools
from typing import Any, Callable


class Pipeline:
    def __init__(self, *fns: Callable):
        self._fns = list(fns)

    def __call__(self, value: Any) -> Any:
        result = value
        for fn in self._fns:
            result = fn(result)
        return result

    def __or__(self, other_fn: Callable) -> "Pipeline":
        """pipeline | fn  →  new Pipeline with fn appended"""
        return Pipeline(*self._fns, other_fn)

    def map(self, fn: Callable) -> "MappedPipeline":
        return MappedPipeline(*self._fns, _map_fn=fn)

    def filter(self, pred: Callable) -> "Pipeline":
        return Pipeline(*self._fns, lambda xs: [x for x in xs if pred(x)])

    def reduce(self, fn: Callable, initial: Any) -> Any:
        intermediate = self(initial if not self._fns else initial)
        # Run the pipeline first, then reduce
        data = self(initial) if not self._fns else Pipeline(*self._fns)(initial)
        return functools.reduce(fn, data, type(initial)())

    def __repr__(self):
        names = " → ".join(getattr(f, "__name__", repr(f)) for f in self._fns)
        return f"Pipeline({names})"


class MappedPipeline(Pipeline):
    def __init__(self, *fns: Callable, _map_fn: Callable):
        super().__init__(*fns)
        self._map_fn = _map_fn

    def __call__(self, iterable):
        # Run existing pipeline on each element
        base = Pipeline(*self._fns)
        return [self._map_fn(base(item)) for item in iterable]

    def filter(self, pred: Callable) -> "MappedPipeline":
        original_call = self.__call__

        class FilteredMappedPipeline(MappedPipeline):
            def __call__(self, iterable):
                return [x for x in original_call(iterable) if pred(x)]

        return FilteredMappedPipeline(*self._fns, _map_fn=self._map_fn)

    def reduce(self, fn: Callable, initial: Any) -> Any:
        return functools.reduce(fn, self.__call__([initial]) if not isinstance(initial, list) else self(initial), type(initial)())


# --- @pipeline_step decorator ---
import inspect

def pipeline_step(step_name: str = None):
    def decorator(fn: Callable) -> Callable:
        sig = inspect.signature(fn)
        params = list(sig.parameters.values())
        if len(params) != 1:
            raise TypeError(
                f"@pipeline_step functions must accept exactly 1 argument, "
                f"got {len(params)} in {fn.__name__!r}"
            )

        @functools.wraps(fn)
        def wrapper(value):
            return fn(value)

        wrapper.step_name = step_name or fn.__name__
        return wrapper

    return decorator


# --- make_branching_pipeline ---
def make_branching_pipeline(
    condition_fn: Callable,
    true_pipeline: Pipeline,
    false_pipeline: Pipeline,
) -> Callable:
    def branching(value):
        if condition_fn(value):
            return true_pipeline(value)
        return false_pipeline(value)
    branching.__name__ = "branching_pipeline"
    return branching


# --- Demo ---
@pipeline_step("normalize")
def normalize(text: str) -> str:
    import unicodedata
    return unicodedata.normalize("NFKC", text)

@pipeline_step("strip")
def strip_whitespace(text: str) -> str:
    return text.strip()

@pipeline_step("lower")
def to_lower(text: str) -> str:
    return text.lower()

@pipeline_step("tokenize")
def tokenize(text: str) -> list[str]:
    return text.split()

@pipeline_step("merge")
def merge_tokens(acc: str, tok: str) -> str:
    return acc + (" " if acc else "") + tok


# Build and run a pipeline
p = Pipeline(normalize, strip_whitespace) | to_lower

print(p("  Hello  WORLD  "))   # hello  world
print(repr(p))                  # Pipeline(normalize → strip → to_lower)

# map + filter
texts = ["  Hello World  ", "  hi  ", "  FOO BAR BAZ  "]
result = (
    Pipeline(normalize, strip_whitespace)
    | to_lower
).map(tokenize).filter(lambda tokens: len(tokens) > 1)

print(result(texts))   # [['hello', 'world'], ['foo', 'bar', 'baz']]


# Branching pipeline
upper_pipeline = Pipeline(str.upper)
lower_pipeline = Pipeline(str.lower)

route = make_branching_pipeline(
    condition_fn=lambda s: s.startswith("A"),
    true_pipeline=upper_pipeline,
    false_pipeline=lower_pipeline,
)

words = ["Alice", "Bob", "Andrew", "Charlie"]
print([route(w) for w in words])   # ['ALICE', 'bob', 'ANDREW', 'charlie']
```
