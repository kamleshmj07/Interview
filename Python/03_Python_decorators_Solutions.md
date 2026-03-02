# Python Decorators — Solutions

## Exercise 1 — Basic Decorator Structure

```python
def trace(fn):
    calls = [0]

    def wrapper(*args, **kwargs):
        calls[0] += 1
        print(f"[TRACE] Entering {fn.__name__} | call #{calls[0]}")
        result = fn(*args, **kwargs)
        print(f"[TRACE] Exiting  {fn.__name__} | returned {result!r}")
        return result

    wrapper.call_count = calls
    return wrapper


def handle_get(method: str, path: str) -> dict:
    return {"status": 200, "method": method, "path": path}

def handle_post(method: str, path: str) -> dict:
    return {"status": 201, "method": method, "path": path}

def handle_delete(method: str, path: str) -> dict:
    return {"status": 204, "method": method, "path": path}


traced_handler = trace(handle_get)
traced_handler("GET", "/users")
traced_handler("GET", "/orders")
print(f"call_count: {traced_handler.call_count[0]}")   # 2

# Original is untouched
print(handle_get("GET", "/users"))

# Inspect closure — fn and calls are captured
for cell in traced_handler.__closure__:
    try:
        print(f"  closure cell: {cell.cell_contents!r}")
    except ValueError:
        pass
```

---

## Exercise 2 — `@` Syntax

```python
import functools


def trace(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        print(f"[TRACE] {fn.__name__} called")
        return fn(*args, **kwargs)
    return wrapper


def mark_as_job(fn):
    fn.is_job = True
    return fn   # mutates and returns original — no wrapper


@trace
def sync_users(): return "synced"

@trace
def send_emails(): return "sent"

@mark_as_job
@trace
def generate_report(): return "report"

@mark_as_job
@trace
def archive_logs(): return "archived"


# @ is exactly equivalent to manual assignment
def dummy(): pass
manually = trace(dummy)
print(manually.__name__ == "dummy")   # True


# job_registry — collect only marked functions
all_jobs = [sync_users, send_emails, generate_report, archive_logs]
job_registry = [fn for fn in all_jobs if getattr(fn, "is_job", False)]
print([fn.__name__ for fn in job_registry])   # ['generate_report', 'archive_logs']


# Programmatic stacking via functools.reduce
def apply_decorators(fn, decorators):
    """Applies decorators right-to-left — equivalent to top-to-bottom @ stacking."""
    return functools.reduce(lambda f, d: d(f), reversed(decorators), fn)

def bold(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        return f"**{fn(*args, **kwargs)}**"
    return wrapper

def upper(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        return fn(*args, **kwargs).upper()
    return wrapper

def greet(name): return f"hello {name}"

decorated = apply_decorators(greet, [bold, upper])
print(decorated("alice"))   # **HELLO ALICE**
```

---

## Exercise 3 — Decorators with `*args` and `**kwargs`

```python
import functools


def log_call(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        print(f"[LOG] {fn.__name__}(args={args}, kwargs={kwargs})")
        result = fn(*args, **kwargs)
        print(f"[LOG] {fn.__name__} -> {result!r}")
        return result
    return wrapper


@log_call
def ingest(source: str, batch_size: int = 100) -> int:
    return batch_size

@log_call
def transform(data: list, *, schema: str, strict: bool = False) -> dict:
    return {"schema": schema, "rows": len(data), "strict": strict}

@log_call
def export(dest: str, *formats, overwrite: bool = False) -> bool:
    return True


ingest("s3://bucket/file.csv")
ingest("gcs://bucket/", batch_size=500)
transform([1, 2, 3], schema="v2", strict=True)
export("/tmp/out", "csv", "parquet", overwrite=True)


def enforce_positional_only(n: int):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            if len(args) < n:
                raise TypeError(
                    f"{fn.__name__}() requires at least {n} positional "
                    f"argument(s), got {len(args)}"
                )
            return fn(*args, **kwargs)
        return wrapper
    return decorator


def forbid_kwargs(*names):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            forbidden = [k for k in names if k in kwargs]
            if forbidden:
                raise TypeError(
                    f"{fn.__name__}() received forbidden kwargs: {forbidden}"
                )
            return fn(*args, **kwargs)
        return wrapper
    return decorator


@enforce_positional_only(2)
def connect(host, port, timeout=30):
    return f"{host}:{port}"

connect("localhost", 5432)
try:
    connect("localhost")
except TypeError as e:
    print(e)

@forbid_kwargs("password", "secret")
def login(username, **kwargs):
    return username

try:
    login("alice", password="hunter2")
except TypeError as e:
    print(e)

# Why wrapper(*args, **kwargs) is the only safe pattern:
# Hard-coding a signature like wrapper(a, b) would fail for functions
# with keyword-only args, *args, or **kwargs — the decorator becomes
# signature-specific and loses its generality.
```

---

## Exercise 4 — Preserving Metadata (`functools.wraps`)

```python
import functools
import inspect


# --- Without functools.wraps ---
def sanitize_broken(fn):
    def wrapper(*args, **kwargs):
        return fn(*args, **kwargs)
    return wrapper


def process_payment(amount: float, currency: str = "USD") -> dict:
    """Process a payment transaction.

    Args:
        amount: Transaction amount in base units.
        currency: ISO 4217 currency code.
    Returns:
        Transaction receipt as a dict.
    """
    return {"amount": amount, "currency": currency}

process_payment.team = "payments"

broken = sanitize_broken(process_payment)
print(broken.__name__)         # wrapper        WRONG
print(broken.__doc__)          # None           WRONG
print(broken.__annotations__)  # {}             WRONG
print(broken.__qualname__)     # sanitize_broken.<locals>.wrapper  WRONG
print(broken.__dict__)         # {}             custom attrs lost


# --- With functools.wraps ---
def sanitize(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        return fn(*args, **kwargs)
    return wrapper

decorated = sanitize(process_payment)
print(decorated.__name__)         # process_payment  correct
print(decorated.__doc__[:28])     # Process a payment  correct
print(decorated.__annotations__)  # {'amount': float, ...}  correct
print(decorated.__qualname__)     # process_payment  correct
print(decorated.__dict__)         # {'team': 'payments'}  correct — __dict__ copied
print(decorated.__wrapped__)      # <function process_payment ...>  set by wraps


# --- unwrap utility ---
def unwrap(fn):
    while hasattr(fn, "__wrapped__"):
        fn = fn.__wrapped__
    return fn

print(unwrap(decorated) is process_payment)   # True


# --- inspect.signature follows __wrapped__ ---
sig = inspect.signature(decorated)
print(sig)   # (amount: float, currency: str = 'USD') -> dict
for name, param in sig.parameters.items():
    print(f"  {name}: annotation={param.annotation}, default={param.default}")
```

---

## Exercise 5 — Stacked / Multiple Decorators

```python
import functools
import time


class RateLimitError(Exception): pass


def require_auth(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        token = kwargs.get("token")
        if not token:
            raise PermissionError(f"[require_auth] {fn.__name__}: missing token")
        return fn(*args, **kwargs)
    return wrapper


def rate_limit(max_calls: int, period: float):
    def decorator(fn):
        calls = []
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            now = time.monotonic()
            while calls and now - calls[0] >= period:
                calls.pop(0)
            if len(calls) >= max_calls:
                raise RateLimitError(f"[rate_limit] {fn.__name__}: exceeded {max_calls} calls")
            calls.append(now)
            return fn(*args, **kwargs)
        return wrapper
    return decorator


def audit(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        user = kwargs.get("user", "anonymous")
        print(f"[AUDIT] {fn.__name__} called by '{user}'")
        return fn(*args, **kwargs)
    return wrapper


# Stack: @audit -> @rate_limit(5,60) -> @require_auth (top to bottom)
# Call stack when invoked:
#   audit.wrapper
#     -> rate_limit.wrapper
#       -> require_auth.wrapper
#         -> get_dashboard (original)
@audit
@rate_limit(5, 60)
@require_auth
def get_dashboard(user: str, token: str) -> dict:
    """Returns the user dashboard."""
    return {"dashboard": "data", "user": user}


print(get_dashboard(user="alice", token="secret"))

# Navigate wrapper chain
print(get_dashboard.__wrapped__.__wrapped__.__name__)  # get_dashboard


# Swap order: @require_auth on top — now it gates before @audit logs
@require_auth
@audit
def get_dashboard_v2(user: str, token: str) -> dict:
    return {"dashboard": "data"}

# No token: require_auth fires FIRST — audit never logs
try:
    get_dashboard_v2(user="alice", token="")
except PermissionError as e:
    print(e)

# Original order: audit logs BEFORE require_auth blocks
try:
    get_dashboard(user="alice", token="")
except PermissionError as e:
    print(e)   # [AUDIT] get_dashboard called by 'alice'  ... then PermissionError


# --- decorator_stack utility ---
def decorator_stack(*decorators):
    """Apply decorators top-to-bottom, matching @ stacking order."""
    def apply(fn):
        return functools.reduce(lambda f, d: d(f), reversed(decorators), fn)
    return apply

@decorator_stack(audit, rate_limit(3, 10), require_auth)
def get_profile(user: str, token: str) -> dict:
    return {"profile": user}

print(get_profile(user="bob", token="xyz"))
```

---

## Exercise 6 — Class Decorators

```python
import functools
import inspect
from datetime import datetime


PLUGIN_REGISTRY: dict = {}


class FrozenError(Exception): pass


def register_plugin(cls):
    if not callable(getattr(cls, "run", None)):
        raise TypeError(f"Plugin '{cls.__name__}' must define a run(self) method")
    cls.registered_at = datetime.utcnow()
    PLUGIN_REGISTRY[cls.__name__] = cls
    return cls   # returns SAME object — no wrapper


@register_plugin
class AudioPlugin:
    def run(self): return "audio running"

@register_plugin
class VideoPlugin:
    def run(self): return "video running"

@register_plugin
class NetworkPlugin:
    def run(self): return "network running"

print(list(PLUGIN_REGISTRY.keys()))   # ['AudioPlugin', 'VideoPlugin', 'NetworkPlugin']
print(AudioPlugin.registered_at)

try:
    @register_plugin
    class BrokenPlugin: pass
except TypeError as e:
    print(e)


def freeze(cls):
    original_init = cls.__init__

    @functools.wraps(original_init)
    def new_init(self, *args, **kwargs):
        original_init(self, *args, **kwargs)
        object.__setattr__(self, "_frozen", True)

    def frozen_setattr(self, name, value):
        if object.__getattribute__(self, "_frozen"):
            raise FrozenError(f"Cannot set '{name}' — instance is frozen")
        object.__setattr__(self, name, value)

    cls.__init__ = new_init
    cls.__setattr__ = frozen_setattr
    return cls


@freeze
class Config:
    def __init__(self, host: str, port: int):
        self.host = host
        self.port = port

cfg = Config("localhost", 5432)
print(cfg.host)
try:
    cfg.host = "newhost"
except FrozenError as e:
    print(e)


def add_repr(cls):
    init_sig = inspect.signature(cls.__init__)
    param_names = [n for n in init_sig.parameters if n != "self"]

    def __repr__(self):
        attrs = ", ".join(f"{p}={getattr(self, p)!r}" for p in param_names)
        return f"{type(self).__name__}({attrs})"

    cls.__repr__ = __repr__
    return cls


# Composing all three
@add_repr
@freeze
@register_plugin
class ImagePlugin:
    def __init__(self, codec: str, resolution: str):
        self.codec = codec
        self.resolution = resolution

    def run(self):
        return f"image: {self.codec} @ {self.resolution}"

img = ImagePlugin("h264", "1080p")
print(repr(img))   # ImagePlugin(codec='h264', resolution='1080p')
try:
    img.codec = "av1"
except FrozenError as e:
    print(e)
```

---

## Exercise 7 — Classes as Decorators

```python
import functools
import time


class ThrottleError(Exception): pass


class Throttle:
    def __init__(self, fn):
        functools.update_wrapper(self, fn)
        self._fn = fn
        self._call_log: list = []

    def __call__(self, *args, **kwargs):
        self._call_log.append(time.monotonic())
        return self._fn(*args, **kwargs)

    def stats(self) -> dict:
        if not self._call_log:
            return {"total_calls": 0, "first_call": None, "last_call": None}
        return {
            "total_calls": len(self._call_log),
            "first_call":  self._call_log[0],
            "last_call":   self._call_log[-1],
        }


@Throttle
def fetch_prices(symbol: str) -> dict:
    """Fetch current prices for a symbol."""
    return {"symbol": symbol, "price": 42.0}

@Throttle
def fetch_orders(account_id: int) -> list:
    return [{"id": 1}]


fetch_prices("AAPL")
fetch_prices("TSLA")
print(fetch_prices.stats())

print(isinstance(fetch_prices, Throttle))   # True — it's an instance
print(fetch_prices.__name__)                # fetch_prices (set by update_wrapper)
print(fetch_prices.__doc__)                 # Fetch current prices...


# ThrottleWithLimit: two-phase __call__
# Phase 1: ThrottleWithLimit(max_calls=3) -> stores config, self is returned
# Phase 2: self(fn) -> receives the function, returns self as the callable wrapper
class ThrottleWithLimit:
    def __init__(self, max_calls: int):
        self.max_calls = max_calls
        self._fn = None
        self._call_log: list = []

    def __call__(self, fn_or_val, *args, **kwargs):
        if self._fn is None and callable(fn_or_val) and not args and not kwargs:
            # Phase 2 — receiving the function
            self._fn = fn_or_val
            functools.update_wrapper(self, fn_or_val)
            return self
        # Phase 3 — normal invocation
        if len(self._call_log) >= self.max_calls:
            raise ThrottleError(
                f"{self._fn.__name__}: exceeded {self.max_calls} calls"
            )
        self._call_log.append(time.monotonic())
        return self._fn(fn_or_val, *args, **kwargs)


@ThrottleWithLimit(max_calls=3)
def fetch_inventory(warehouse_id: int) -> list:
    return [{"item": "widget"}]

fetch_inventory(1)
fetch_inventory(2)
fetch_inventory(3)
try:
    fetch_inventory(4)
except ThrottleError as e:
    print(e)

# Class vs. closure:
# Class advantages:
#   1. Clean state management with named attributes and helper methods (stats()).
#   2. Supports subclassing — ThrottleWithLimit(Throttle) is natural.
# Class disadvantages:
#   1. isinstance returns the class, not a function — may surprise inspect/functools tools.
#   2. Must call functools.update_wrapper manually; @functools.wraps won't work on __call__.
# Closure advantages:
#   1. Returns a real function — transparent to all standard tooling.
#   2. Less boilerplate for simple cases.
# Closure disadvantages:
#   1. No clean way to expose helper methods without attaching to wrapper.__dict__.
#   2. Harder to subclass or extend decorator behaviour without rewriting the closure.
```

---

## Exercise 8 — Decorator Factories

```python
import functools
import time
import warnings


class ValidationError(Exception): pass


# Three-level structure:
# Level 1 — factory(params):        configures and returns the decorator
# Level 2 — decorator(fn):          wraps the target function
# Level 3 — wrapper(*args,**kwargs): executes at each call site


def validate(schema: dict):                          # Level 1
    def decorator(fn):                               # Level 2
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):                # Level 3
            import inspect as _inspect
            bound = _inspect.signature(fn).bind(*args, **kwargs)
            bound.apply_defaults()
            for param, (expected_type, validator_fn) in schema.items():
                if param not in bound.arguments:
                    continue
                val = bound.arguments[param]
                if not isinstance(val, expected_type):
                    raise ValidationError(
                        f"'{param}' expected {expected_type.__name__}, "
                        f"got {type(val).__name__}"
                    )
                if not validator_fn(val):
                    raise ValidationError(
                        f"'{param}' failed validation: {val!r}"
                    )
            return fn(*args, **kwargs)
        return wrapper
    return decorator


def transform_output(transform_fn):                  # Level 1
    def decorator(fn):                               # Level 2
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):                # Level 3
            return transform_fn(fn(*args, **kwargs))
        return wrapper
    return decorator


def deprecated(reason: str, replacement: str = None):   # Level 1
    def decorator(fn):                                    # Level 2
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):                     # Level 3
            msg = f"{fn.__name__} is deprecated: {reason}"
            if replacement:
                msg += f" — use '{replacement}' instead"
            warnings.warn(msg, DeprecationWarning, stacklevel=2)
            return fn(*args, **kwargs)
        return wrapper
    return decorator


def retry(max_attempts=3, delay=0.0, exceptions=(Exception,), on_failure=None):  # Level 1
    def decorator(fn):                                                             # Level 2
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):                                              # Level 3
            last_exc = None
            for attempt in range(1, max_attempts + 1):
                try:
                    return fn(*args, **kwargs)
                except exceptions as e:
                    last_exc = e
                    if attempt < max_attempts:
                        time.sleep(delay)
            if on_failure:
                on_failure(fn, max_attempts, last_exc)
            raise last_exc
        return wrapper
    return decorator


def failure_cb(fn, attempts, exc):
    print(f"[FAILURE] {fn.__name__} gave up after {attempts} attempt(s): {exc}")

@validate({"amount": (float, lambda x: x > 0), "currency": (str, lambda s: len(s) == 3)})
@transform_output(lambda r: {**r, "formatted": f"${r['amount']:.2f}"})
@retry(max_attempts=2, delay=0.0, on_failure=failure_cb)
def create_charge(amount: float, currency: str = "USD") -> dict:
    """Create a payment charge."""
    return {"amount": amount, "currency": currency, "status": "charged"}

print(create_charge(49.99, "USD"))

try:
    create_charge(-10.0, "USD")
except ValidationError as e:
    print(e)

@deprecated("Use create_charge() instead", replacement="create_charge")
def old_charge(amount): return amount

import warnings as _w
with _w.catch_warnings(record=True) as w:
    _w.simplefilter("always")
    old_charge(10.0)
    print(w[0].message)
```

---

## Exercise 9 — Common Use Cases

```python
import functools
import time
import json
from datetime import datetime, timezone


class AuthorizationError(Exception): pass


def timer(verbose: bool = True):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            t0 = time.perf_counter()
            result = fn(*args, **kwargs)
            duration = time.perf_counter() - t0
            wrapper.last_duration = duration
            if verbose:
                print(f"[TIMER] {fn.__name__} took {duration*1000:.3f}ms")
            return result
        wrapper.last_duration = None
        return wrapper
    return decorator


def structured_log(level: str = "INFO"):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            t0 = time.perf_counter()
            result = fn(*args, **kwargs)
            entry = {
                "level": level,
                "fn_name": fn.__name__,
                "args": args,
                "kwargs": kwargs,
                "result": result,
                "duration_ms": round((time.perf_counter() - t0) * 1000, 3),
                "timestamp": datetime.now(timezone.utc).isoformat(),
            }
            print(json.dumps(entry, default=str))
            return result
        return wrapper
    return decorator


def memoize(fn):
    cache: dict = {}
    hits = [0]
    misses = [0]

    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        key = (args, frozenset(kwargs.items()))
        if key in cache:
            hits[0] += 1
            return cache[key]
        misses[0] += 1
        cache[key] = fn(*args, **kwargs)
        return cache[key]

    wrapper.cache_info  = lambda: {"hits": hits[0], "misses": misses[0], "size": len(cache)}
    wrapper.cache_clear = lambda: (cache.clear(), hits.__setitem__(0, 0), misses.__setitem__(0, 0))
    return wrapper


@memoize
def fibonacci(n: int) -> int:
    if n < 2: return n
    return fibonacci(n - 1) + fibonacci(n - 2)

fibonacci(30)
print(fibonacci.cache_info())


@memoize
def expensive_query(record_id: int, filters=None) -> dict:
    return {"id": record_id, "filters": filters}

expensive_query(1, filters="active")
expensive_query(1, filters="active")   # cache hit
print(expensive_query.cache_info())


def require_roles(*roles):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            user = kwargs.get("current_user", {})
            user_roles = set(user.get("roles", []))
            if not user_roles.intersection(roles):
                raise AuthorizationError(
                    f"{fn.__name__}: user roles {user_roles} "
                    f"do not include any of {set(roles)}"
                )
            return fn(*args, **kwargs)
        return wrapper
    return decorator


# Stacking order justification (top to bottom = outermost to innermost):
#   @memoize        — cache at the outermost public interface
#   @structured_log — log every non-cached call with timing
#   @timer          — measure business logic duration
#   @require_roles  — gate first: fail fast before any work
@memoize
@structured_log(level="INFO")
@timer(verbose=False)
@require_roles("admin", "superuser")
def get_report(report_id: int, current_user: dict) -> dict:
    """Generate and return a report by ID."""
    return {"report_id": report_id, "data": "..."}

admin = {"id": 1, "roles": ["admin"]}
print(get_report(report_id=42, current_user=admin))

try:
    get_report(report_id=42, current_user={"id": 2, "roles": ["viewer"]})
except AuthorizationError as e:
    print(e)
```

---

## Exercise 10 — Attribute Preservation

```python
import functools
import inspect


def process_order(order_id: int, items: list, discount: float = 0.0) -> dict:
    """Process a customer order.

    Validates the order, applies discounts, and submits for fulfillment.

    Args:
        order_id: Unique order identifier.
        items: List of item dicts with 'sku' and 'qty'.
        discount: Fractional discount to apply (0.0-1.0).
    Returns:
        Fulfillment receipt dict.
    """
    return {"order_id": order_id}

process_order.version = "3.1"
process_order.tags = ["orders", "fulfillment"]
process_order.owner = "commerce-team"


def decorator_a(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        return fn(*args, **kwargs)
    return wrapper

def decorator_b(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        return fn(*args, **kwargs)
    return wrapper

def decorator_c_broken(fn):
    def wrapper(*args, **kwargs):   # no functools.wraps
        return fn(*args, **kwargs)
    return wrapper

def decorator_c_fixed(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        return fn(*args, **kwargs)
    return wrapper


# Broken stack — decorator_c_broken destroys the chain
@decorator_a
@decorator_c_broken
@decorator_b
def broken_fn(order_id: int) -> dict:
    """Should survive."""
    return {}
broken_fn.version = "1.0"

print(broken_fn.__name__)   # wrapper   BROKEN
print(broken_fn.__doc__)    # None      BROKEN
# __wrapped__ from decorator_a exists but its target is decorator_c's unwrapped wrapper
print(broken_fn.__wrapped__.__name__)   # wrapper  still broken at the c layer


# Fixed stack — all layers use functools.wraps
@decorator_a
@decorator_c_fixed
@decorator_b
def good_fn(order_id: int) -> dict:
    """Should survive."""
    return {}
good_fn.version = "3.1"
good_fn.tags = ["orders"]

print(good_fn.__name__)     # good_fn  correct
print(good_fn.__dict__)     # {'version': '3.1', 'tags': [...]}  correct
print(good_fn.__wrapped__.__wrapped__.__name__)   # good_fn  navigable


# assert_metadata_intact helper
def assert_metadata_intact(original, decorated):
    attrs = ["__name__", "__doc__", "__annotations__", "__qualname__", "__module__"]
    errors = []
    for attr in attrs:
        if getattr(original, attr, None) != getattr(decorated, attr, None):
            errors.append(
                f"{attr}: expected {getattr(original, attr, None)!r}, "
                f"got {getattr(decorated, attr, None)!r}"
            )
    for key, val in original.__dict__.items():
        if getattr(decorated, key, None) != val:
            errors.append(f"__dict__[{key!r}]: missing or wrong")
    if errors:
        raise AssertionError("Metadata mismatch:\n" + "\n".join(f"  {e}" for e in errors))
    print("All metadata intact")

assert_metadata_intact(good_fn, good_fn)


# inspect.signature follows __wrapped__ — correct
@decorator_a
def annotated(x: int, y: str = "hello") -> bool:
    """Annotated function."""
    return True

print(inspect.signature(annotated))   # (x: int, y: str = 'hello') -> bool

# Without __wrapped__, signature degrades
def deco_no_wraps(fn):
    def wrapper(*args, **kwargs):
        return fn(*args, **kwargs)
    return wrapper

print(inspect.signature(deco_no_wraps(annotated)))   # (*args, **kwargs)  degraded
```

---

## Bonus Challenge — `pipeline_decorator` & `DecoratorRegistry`

```python
import functools
import time
import inspect
from datetime import datetime


# Reusable decorators

def require_auth(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        if not kwargs.get("token"):
            raise PermissionError(f"{fn.__name__}: missing token")
        return fn(*args, **kwargs)
    return wrapper


def rate_limit(max_calls: int, period: float):
    def decorator(fn):
        calls = []
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            now = time.monotonic()
            while calls and now - calls[0] >= period:
                calls.pop(0)
            if len(calls) >= max_calls:
                raise RuntimeError(f"{fn.__name__}: rate limit exceeded")
            calls.append(now)
            return fn(*args, **kwargs)
        return wrapper
    return decorator


def structured_log(level="INFO"):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            result = fn(*args, **kwargs)
            print(f"[{level}] {fn.__name__} -> {result!r}")
            return result
        return wrapper
    return decorator


def timer(verbose=False):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            t0 = time.perf_counter()
            result = fn(*args, **kwargs)
            if verbose:
                print(f"[TIMER] {fn.__name__}: {(time.perf_counter()-t0)*1000:.2f}ms")
            return result
        return wrapper
    return decorator


# DecoratorRegistry

class DecoratorRegistry:
    def __init__(self):
        self._registry: dict = {}

    def track(self, fn):
        """Decorator: registers fn into this registry."""
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            return fn(*args, **kwargs)
        self._registry[fn.__name__] = {
            "fn":          fn,
            "registered":  datetime.utcnow().isoformat(),
            "__name__":    fn.__name__,
            "__doc__":     fn.__doc__,
            "__annotations__": fn.__annotations__,
            "__module__":  fn.__module__,
        }
        return wrapper

    def list_decorated(self) -> list:
        return [(name, list(meta.keys())) for name, meta in self._registry.items()]

    def audit(self, fn_name: str) -> dict:
        if fn_name not in self._registry:
            raise KeyError(f"No function '{fn_name}' in registry")
        meta = self._registry[fn_name]
        return {
            "name":        meta["__name__"],
            "doc":         meta["__doc__"],
            "annotations": meta["__annotations__"],
            "module":      meta["__module__"],
            "registered":  meta["registered"],
            "signature":   str(inspect.signature(meta["fn"])),
        }


# pipeline_decorator

def pipeline_decorator(*decorators, label: str = ""):
    """
    Composes multiple decorators into one reusable decorator.
    Applies them top-to-bottom, matching @ stacking order.
    """
    def combined(fn):
        wrapped = functools.reduce(lambda f, d: d(f), reversed(decorators), fn)
        if not hasattr(wrapped, "__wrapped__"):
            wrapped.__wrapped__ = fn
        if label:
            wrapped._pipeline_label = label
        return wrapped

    combined.__name__ = f"pipeline({label or 'unnamed'})"
    return combined


# Demo

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
    return {"id": user_id, "name": "Alice"}


print(get_user.__name__)   # get_user
print(get_user.__doc__)    # Fetch a user by ID.

result = get_user(user_id=7, token="valid-token")
print(result)

try:
    get_user(user_id=7, token="")
except PermissionError as e:
    print(e)

print(registry.list_decorated())
print(registry.audit("get_user"))


# unwrap to original
def unwrap(fn):
    while hasattr(fn, "__wrapped__"):
        fn = fn.__wrapped__
    return fn

original = unwrap(get_user)
print(original.__name__)      # get_user
print(original is get_user)   # False — original is the bare unwrapped function
```
