# Python OOP — Solutions

---

## Exercise 1 — Classes & Objects/Instances

```python
class StockPosition:
    def __init__(self, ticker: str, shares: float, avg_cost: float):
        self.ticker = ticker
        self.shares = shares
        self.avg_cost = avg_cost  # cost per share

    def current_value(self, price: float) -> float:
        return self.shares * price

    def unrealized_pnl(self, price: float) -> float:
        return (price - self.avg_cost) * self.shares

    def __repr__(self):
        return f"StockPosition(ticker={self.ticker!r}, shares={self.shares}, avg_cost={self.avg_cost})"


# Three distinct instances
aapl = StockPosition("AAPL", 100, 150.00)
tsla = StockPosition("TSLA", 50, 200.00)
msft = StockPosition("MSFT", 200, 300.00)

print(aapl.current_value(180.0))      # 18000.0
print(aapl.unrealized_pnl(180.0))     # 3000.0
print(tsla.unrealized_pnl(180.0))     # -1000.0

# Mutating one does NOT affect others
aapl.shares = 999
print(tsla.shares)   # still 50 — independent objects
```

---

## Exercise 2 — Instance vs. Class Attributes

```python
class DatabaseConnection:
    _dsn: str = "postgresql://localhost/default"
    _pool_size: int = 5

    def __init__(self, connection_id: int):
        self.connection_id = connection_id
        self.query_count = 0

    @classmethod
    def configure(cls, dsn: str, pool_size: int):
        cls._dsn = dsn
        cls._pool_size = pool_size

    def execute(self, sql: str):
        self.query_count += 1
        return f"[{self.connection_id}] Executing: {sql}"


conn1 = DatabaseConnection(1)
conn2 = DatabaseConnection(2)

# Class-level update propagates to all instances
DatabaseConnection.configure("postgresql://prod/db", 20)
print(conn1._dsn)   # postgresql://prod/db
print(conn2._dsn)   # postgresql://prod/db

# Assigning via instance SHADOWS the class attribute
conn1._dsn = "sqlite://local"
print(conn1._dsn)                     # sqlite://local   ← instance shadow
print(conn2._dsn)                     # postgresql://prod/db  ← still class attr
print(DatabaseConnection._dsn)        # postgresql://prod/db  ← class unchanged

# Prove shadowing: conn1 now has its own __dict__ entry
assert "_dsn" in conn1.__dict__
assert "_dsn" not in conn2.__dict__
```

---

## Exercise 3 — Instance, Class & Static Methods

```python
class Temperature:
    def __init__(self, celsius: float):
        self._celsius = celsius

    # --- Instance methods ---
    def to_celsius(self) -> float:
        return self._celsius

    def to_fahrenheit(self) -> float:
        return self._celsius * 9 / 5 + 32

    def to_kelvin(self) -> float:
        return self._celsius + 273.15

    # --- Class methods (factory constructors) ---
    @classmethod
    def from_celsius(cls, value: float) -> "Temperature":
        return cls(value)

    @classmethod
    def from_fahrenheit(cls, value: float) -> "Temperature":
        return cls((value - 32) * 5 / 9)

    @classmethod
    def from_kelvin(cls, value: float) -> "Temperature":
        if not cls.is_valid_kelvin(value):
            raise ValueError(f"Kelvin cannot be negative: {value}")
        return cls(value - 273.15)

    # --- Static method ---
    @staticmethod
    def is_valid_kelvin(value: float) -> bool:
        # Static because it doesn't need class or instance state.
        # It's a pure domain-logic helper that conceptually belongs
        # in the Temperature namespace but has no dependency on self/cls.
        return value >= 0.0

    def __repr__(self):
        return f"Temperature({self._celsius:.4f}°C)"


t = Temperature.from_fahrenheit(212)
print(t.to_celsius())    # 100.0
print(t.to_kelvin())     # 373.15

print(Temperature.is_valid_kelvin(-1))   # False (via class)
print(t.is_valid_kelvin(300))            # True  (via instance — same method)
```

---

## Exercise 4 — `self` & `__init__`

```python
from typing import Any


class CircularBuffer:
    def __init__(self, capacity: int):
        if capacity < 1:
            raise ValueError("Capacity must be at least 1")
        self._capacity = capacity
        self._buffer: list = [None] * capacity
        self._head = 0   # next write position
        self._tail = 0   # next read position
        self._size = 0

    def push(self, item: Any) -> None:
        self._buffer[self._head] = item
        self._head = (self._head + 1) % self._capacity
        if self._size == self._capacity:
            # Overwrite: advance tail to discard oldest
            self._tail = (self._tail + 1) % self._capacity
        else:
            self._size += 1

    def pop(self) -> Any:
        if self._size == 0:
            raise IndexError("pop from empty CircularBuffer")
        item = self._buffer[self._tail]
        self._tail = (self._tail + 1) % self._capacity
        self._size -= 1
        return item

    def __len__(self) -> int:
        return self._size


buf = CircularBuffer(3)
buf.push(1); buf.push(2); buf.push(3)
buf.push(4)          # overwrites 1
print(buf.pop())     # 2 (oldest remaining)
print(len(buf))      # 2

# Why removing self causes TypeError, not logic error:
# Python passes the instance as the first argument automatically.
# If self is absent, Python tries to pass the instance as the first
# explicit arg, causing: TypeError: push() takes 1 positional argument but 2 were given
```

---

## Exercise 5 — Single & Multiple Inheritance

```python
class Notification:
    def __init__(self, recipient: str, message: str):
        self.recipient = recipient
        self.message = message

    def send(self) -> str:
        raise NotImplementedError(f"{type(self).__name__} must implement send()")


class EmailNotification(Notification):
    def send(self) -> str:
        return f"[EMAIL → {self.recipient}] {self.message}"


class SMSNotification(Notification):
    def send(self) -> str:
        return f"[SMS → {self.recipient}] {self.message}"


class PriorityNotification(EmailNotification, SMSNotification):
    def __init__(self, recipient: str, message: str, priority: int):
        super().__init__(recipient, message)
        self.priority = priority

    def send(self) -> str:
        prefix = f"[PRIORITY-{self.priority}]"
        # MRO ensures EmailNotification.send() is called first
        email_result = EmailNotification.send(self)
        sms_result = SMSNotification.send(self)
        return f"{prefix} {email_result} | {sms_result}"


pn = PriorityNotification("alice@example.com", "Server down", 1)
print(pn.send())

# MRO: PriorityNotification → EmailNotification → SMSNotification → Notification → object
print(PriorityNotification.__mro__)
# Diamond resolution: C3 linearization ensures Notification appears only once,
# after all its subclasses. super() cooperatively calls the next in MRO.
```

---

## Exercise 6 — `super()`

```python
class Employee:
    def __init__(self, name: str, base_salary: float):
        self.name = name
        self.base_salary = base_salary

    def annual_compensation(self) -> float:
        return self.base_salary


class Manager(Employee):
    def __init__(self, name: str, base_salary: float, team_size: int):
        super().__init__(name, base_salary)
        self.team_size = team_size

    def annual_compensation(self) -> float:
        return super().annual_compensation() + self.team_size * 5_000


class Director(Manager):
    def __init__(self, name: str, base_salary: float, team_size: int, budget: float):
        super().__init__(name, base_salary, team_size)
        self.budget = budget

    def annual_compensation(self) -> float:
        # super() here calls Manager.annual_compensation(), which in turn
        # calls Employee.annual_compensation() via its own super()
        return super().annual_compensation() + self.budget * 0.01


e = Employee("Alice", 60_000)
m = Manager("Bob", 80_000, 6)
d = Director("Carol", 120_000, 10, 5_000_000)

print(e.annual_compensation())   # 60000
print(m.annual_compensation())   # 110000  (80000 + 6*5000)
print(d.annual_compensation())   # 220000  (120000 + 10*5000 + 5000000*0.01)

# If you remove super() in Director.annual_compensation() and hard-code
# Employee.annual_compensation(self), you lose the Manager bonus silently.
```

---

## Exercise 7 — Encapsulation

```python
class InsufficientFundsError(Exception):
    pass


class BankAccount:
    def __init__(self, owner: str, initial_balance: float = 0.0):
        self.owner = owner
        self.__balance = initial_balance          # name-mangled → _BankAccount__balance
        self._transaction_log: list[str] = []    # protected by convention

    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Deposit amount must be positive")
        self.__balance += amount
        self._transaction_log.append(f"DEPOSIT +{amount:.2f}")

    def withdraw(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Withdrawal amount must be positive")
        if amount > self.__balance:
            raise InsufficientFundsError(
                f"Cannot withdraw {amount:.2f}; balance is {self.__balance:.2f}"
            )
        self.__balance -= amount
        self._transaction_log.append(f"WITHDRAW -{amount:.2f}")

    @property
    def balance(self) -> float:
        return self.__balance


acct = BankAccount("Alice", 1000)
acct.deposit(500)
acct.withdraw(200)

# Name mangling in action
try:
    print(acct.__balance)         # AttributeError — as intended
except AttributeError as e:
    print(f"Blocked: {e}")

print(acct._BankAccount__balance) # 1300.0 — accessible but signals "don't touch"
# This is a FEATURE: it prevents accidental overrides in subclasses,
# not a security mechanism.

print(acct._transaction_log)      # accessible — "internal but not forbidden"
```

---

## Exercise 8 — Polymorphism: Duck Typing & Method Overriding

```python
import json
import csv
import io

# --- Part 1: Duck typing (no shared base) ---
class PDFExporter:
    def export(self, data: dict) -> str:
        return f"[PDF BINARY] keys={list(data.keys())}"

class CSVExporter:
    def export(self, data: dict) -> str:
        out = io.StringIO()
        writer = csv.writer(out)
        writer.writerow(data.keys())
        writer.writerow(data.values())
        return out.getvalue().strip()

class JSONExporter:
    def export(self, data: dict) -> str:
        return json.dumps(data, indent=2)

def run_export(exporter, data: dict) -> str:
    # No isinstance check — relies purely on duck typing
    return exporter.export(data)

sample = {"name": "Alice", "age": 30}
for exp in [PDFExporter(), CSVExporter(), JSONExporter()]:
    print(run_export(exp, sample))

# --- Part 2: Formal base class + method overriding ---
class BaseExporter:
    def export(self, data: dict) -> str:
        raise NotImplementedError

class CompressedCSVExporter(CSVExporter):
    def export(self, data: dict) -> str:
        raw = super().export(data)
        # Mock compression
        compressed = f"[GZIP:{len(raw.encode())}bytes] {raw[:20]}..."
        return compressed

print(CompressedCSVExporter().export(sample))

# Duck typing = structural (works if it walks like a duck)
# Method overriding = nominal (subclass explicitly replaces parent behavior)
# Duck typing has less coupling; method overriding provides a type contract.
```

---

## Exercise 9 — Abstract Base Classes

```python
import re
from abc import ABC, abstractmethod


class DataValidator(ABC):
    @abstractmethod
    def validate(self, value) -> bool: ...

    @abstractmethod
    def error_message(self) -> str: ...

    def check(self, value) -> str:
        if self.validate(value):
            return f"✓ '{value}' is valid"
        return f"✗ '{value}' failed: {self.error_message()}"


class EmailValidator(DataValidator):
    def validate(self, value) -> bool:
        return bool(re.match(r"[^@]+@[^@]+\.[^@]+", value))

    def error_message(self) -> str:
        return "Not a valid email address"


class PhoneValidator(DataValidator):
    def validate(self, value) -> bool:
        return bool(re.match(r"^\+?[\d\s\-]{7,15}$", value))

    def error_message(self) -> str:
        return "Not a valid phone number"


class AgeValidator(DataValidator):
    def validate(self, value) -> bool:
        return isinstance(value, int) and 0 <= value <= 150

    def error_message(self) -> str:
        return "Age must be an integer between 0 and 150"


# Cannot instantiate ABC
try:
    DataValidator()
except TypeError as e:
    print(f"Cannot instantiate ABC: {e}")

print(EmailValidator().check("alice@example.com"))
print(AgeValidator().check(200))

# --- Virtual subclassing ---
class ThirdPartyChecker:
    """External class we cannot modify."""
    def validate(self, value) -> bool:
        return value is not None

    def error_message(self) -> str:
        return "Value is None"

DataValidator.register(ThirdPartyChecker)

checker = ThirdPartyChecker()
print(isinstance(checker, DataValidator))   # True — virtual subclass
# But abstract methods are NOT enforced on virtual subclasses.
# Use virtual subclassing when you want isinstance compatibility
# without modifying third-party code.
```

---

## Exercise 10 — Properties

```python
import math


class Rectangle:
    def __init__(self, width: float, height: float):
        self.width = width    # goes through setter
        self.height = height

    @property
    def width(self) -> float:
        return self._width

    @width.setter
    def width(self, value: float):
        if value <= 0:
            raise ValueError("Width must be positive")
        self._width = value

    @width.deleter
    def width(self):
        self._width = 1.0   # reset to default

    @property
    def height(self) -> float:
        return self._height

    @height.setter
    def height(self, value: float):
        if value <= 0:
            raise ValueError("Height must be positive")
        self._height = value

    @property
    def area(self) -> float:
        return self._width * self._height

    @property
    def perimeter(self) -> float:
        return 2 * (self._width + self._height)

    @property
    def diagonal(self) -> float:
        return math.hypot(self._width, self._height)

    @property
    def scale(self) -> float:
        raise AttributeError("scale is write-only")

    @scale.setter
    def scale(self, factor: float):
        if factor <= 0:
            raise ValueError("Scale factor must be positive")
        self._width *= factor
        self._height *= factor


r = Rectangle(4, 3)
print(r.area)         # 12
print(r.diagonal)     # 5.0
r.scale = 2
print(r.area)         # 48

del r.width           # resets to 1.0
print(r.width)        # 1.0

try:
    r.area = 100      # AttributeError — no setter on area
except AttributeError as e:
    print(e)
```

---

## Exercise 11 — `@classmethod` Deep Dive

```python
import json
import os
from typing import Any


class Config:
    _DEFAULTS = {"debug": False, "log_level": "INFO", "timeout": 30}

    def __init__(self, data: dict):
        self._data = data

    @classmethod
    def from_json(cls, json_str: str) -> "Config":
        return cls(json.loads(json_str))

    @classmethod
    def from_env(cls, prefix: str) -> "Config":
        data = {
            k[len(prefix):].lower(): v
            for k, v in os.environ.items()
            if k.startswith(prefix)
        }
        return cls(data)

    @classmethod
    def default(cls) -> "Config":
        return cls(dict(cls._DEFAULTS))

    def get(self, key: str, fallback: Any = None) -> Any:
        return self._data.get(key, fallback)

    def __repr__(self):
        return f"{type(self).__name__}({self._data})"


class AppConfig(Config):
    _DEFAULTS = {**Config._DEFAULTS, "app_name": "MyApp", "version": "1.0.0"}

    @classmethod
    def default(cls) -> "AppConfig":
        return cls(dict(cls._DEFAULTS))


cfg = AppConfig.default()
print(cfg)                     # AppConfig({...}) — NOT Config
print(type(cfg).__name__)      # AppConfig

# cls vs. hard-coding: if default() used Config(...) instead of cls(...),
# AppConfig.default() would return a Config instance, losing subclass type.
```

---

## Exercise 12 — `@staticmethod` Deep Dive

```python
class MathUtils:
    @staticmethod
    def clamp(value: float, min_val: float, max_val: float) -> float:
        return max(min_val, min(max_val, value))

    @staticmethod
    def lerp(a: float, b: float, t: float) -> float:
        return a + t * (b - a)

    @staticmethod
    def is_prime(n: int) -> bool:
        if n < 2:
            return False
        for i in range(2, int(n ** 0.5) + 1):
            if n % i == 0:
                return False
        return True

    @staticmethod
    def fibonacci(n: int) -> list[int]:
        if n <= 0:
            return []
        seq = [0, 1]
        while len(seq) < n:
            seq.append(seq[-1] + seq[-2])
        return seq[:n]


# Call on class or instance — identical behavior
print(MathUtils.clamp(15, 0, 10))         # 10
mu = MathUtils()
print(mu.is_prime(97))                    # True
print(mu.fibonacci(8))

# Static method has NO access to cls or self:
# @staticmethod cannot call cls._DEFAULTS or self._data
# This is intentional — it enforces that the logic is truly stateless.

# When to use staticmethod vs. module-level function:
# - Use @staticmethod when the function is CONCEPTUALLY part of the class namespace
#   (e.g., MathUtils.clamp makes the grouping clear).
# - Use module-level functions when the function is general-purpose utility
#   with no conceptual tie to a specific class.
# - Avoid @staticmethod if the function is also useful outside the class context.
```

---

## Exercise 13 — `__str__` and `__repr__`

```python
import math


class Vector3D:
    def __init__(self, x: float, y: float, z: float):
        self.x = x
        self.y = y
        self.z = z

    def __repr__(self) -> str:
        # eval()-able: Vector3D(x=1.0, y=2.0, z=3.0)
        return f"Vector3D(x={self.x!r}, y={self.y!r}, z={self.z!r})"

    def __str__(self) -> str:
        return f"<{self.x}, {self.y}, {self.z}>"

    def __format__(self, spec: str) -> str:
        if spec:
            fmt = f"{{:.{spec}}}"  # e.g., ":.2f"
            return f"<{fmt.format(self.x)}, {fmt.format(self.y)}, {fmt.format(self.z)}>"
        return str(self)


v = Vector3D(1.0, 2.0, 3.0)

print(repr(v))          # Vector3D(x=1.0, y=2.0, z=3.0)
print(str(v))           # <1.0, 2.0, 3.0>
print(f"{v}")           # <1.0, 2.0, 3.0>   — uses __str__
print(f"{v!r}")         # Vector3D(x=1.0, y=2.0, z=3.0) — uses __repr__
print([v])              # [Vector3D(x=1.0, y=2.0, z=3.0)] — list uses __repr__
print(f"{v:2f}")        # <1.00, 2.00, 3.00>

# Reconstructable via eval:
v2 = eval(repr(v))
print(v2.x, v2.y, v2.z)   # 1.0 2.0 3.0

# A class with only __repr__: str() falls back to __repr__
# Python's data model: str() → __str__ → fallback to __repr__ if absent
```

---

## Exercise 14 — `__slots__`

```python
import tracemalloc
from datetime import datetime


class Tick:
    __slots__ = ("symbol", "price", "volume", "timestamp")

    def __init__(self, symbol: str, price: float, volume: int, timestamp: datetime):
        self.symbol = symbol
        self.price = price
        self.volume = volume
        self.timestamp = timestamp


class ExtendedTick(Tick):
    __slots__ = ("vwap",)   # Must redeclare in subclass

    def __init__(self, symbol, price, volume, timestamp, vwap: float):
        super().__init__(symbol, price, volume, timestamp)
        self.vwap = vwap


# Prove undeclared attribute raises AttributeError
t = Tick("AAPL", 180.0, 1000, datetime.utcnow())
try:
    t.extra = "oops"
except AttributeError as e:
    print(f"No __dict__: {e}")

# Memory comparison
N = 100_000
now = datetime.utcnow()

tracemalloc.start()
ticks = [Tick("AAPL", 180.0, 1000, now) for _ in range(N)]
snap1 = tracemalloc.take_snapshot()

dicts = [{"symbol": "AAPL", "price": 180.0, "volume": 1000, "timestamp": now} for _ in range(N)]
snap2 = tracemalloc.take_snapshot()
tracemalloc.stop()

stats = snap2.compare_to(snap1, "lineno")
print(f"Extra memory for dicts: {stats[0].size_diff / 1024:.1f} KB")

# Opt back into __dict__ by adding "__dict__" to __slots__:
class FlexibleTick(Tick):
    __slots__ = ("__dict__",)  # re-enables arbitrary attributes
```

---

## Exercise 15 — `__getattr__` / `__setattr__` / `__delattr__`

```python
from __future__ import annotations


class DynamicConfig:
    def __init__(self):
        # MUST use object.__setattr__ to avoid infinite recursion in our __setattr__
        object.__setattr__(self, "_store", {})
        object.__setattr__(self, "_log", [])

    def __getattr__(self, name: str):
        # Only called when normal attribute lookup fails
        try:
            return self._store[name]
        except KeyError:
            raise AttributeError(f"No config key: {name!r}")

    def __setattr__(self, name: str, value):
        # Intercepts ALL attribute assignments
        if name in ("_store", "_log"):
            # Bootstrap: let these through to __dict__
            object.__setattr__(self, name, value)
        else:
            self._store[name] = value
            self._log.append(f"SET {name}={value!r}")

    def __delattr__(self, name: str):
        try:
            del self._store[name]
            self._log.append(f"DEL {name}")
        except KeyError:
            raise AttributeError(f"No config key: {name!r}")

    def __dir__(self):
        return list(super().__dir__()) + list(self._store.keys())


cfg = DynamicConfig()
cfg.host = "localhost"
cfg.port = 5432
print(cfg.host)          # localhost
del cfg.port
print(cfg._log)          # ['SET host=...', 'SET port=...', 'DEL port']
print("port" in dir(cfg))   # False (deleted)
print("host" in dir(cfg))   # True

# Infinite recursion demo (DO NOT run without the guard):
# If __setattr__ did `self._log.append(...)` directly, Python calls __setattr__
# again for `self._log`, causing RecursionError.
# object.__setattr__ bypasses our override and writes directly to __dict__.
```

---

## Exercise 16 — `__call__`

```python
import time
import functools
from collections import deque


class RateLimitExceeded(Exception):
    pass


class RateLimiter:
    def __init__(self, max_calls: int, period: float):
        self.max_calls = max_calls
        self.period = period
        self._calls: deque = deque()

    def check(self) -> bool:
        now = time.monotonic()
        # Purge old timestamps
        while self._calls and now - self._calls[0] >= self.period:
            self._calls.popleft()
        if len(self._calls) >= self.max_calls:
            raise RateLimitExceeded(
                f"Exceeded {self.max_calls} calls in {self.period}s"
            )
        self._calls.append(now)
        return True

    def __call__(self, func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            self.check()
            return func(*args, **kwargs)
        return wrapper

    def __repr__(self):
        return (
            f"RateLimiter(calls={len(self._calls)}/{self.max_calls}, "
            f"period={self.period}s)"
        )


# As a decorator
@RateLimiter(3, 1.0)
def fetch_data(url: str):
    return f"data from {url}"

for _ in range(3):
    print(fetch_data("https://api.example.com"))

try:
    fetch_data("https://api.example.com")   # 4th call in same second
except RateLimitExceeded as e:
    print(e)

# Standalone
limiter = RateLimiter(10, 1.0)
print(callable(limiter))    # True
limiter.check()
print(repr(limiter))
```

---

## Exercise 17 — Method Resolution Order

```python
# Classic diamond
class Base:
    def greet(self): return "Base"

class A(Base):
    def greet(self): return f"A → {super().greet()}"

class B(A):
    def greet(self): return f"B → {super().greet()}"

class C(A):
    def greet(self): return f"C → {super().greet()}"

class D(B, C):
    def greet(self): return f"D → {super().greet()}"

print(D.__mro__)
# D → B → C → A → Base → object
# C3 linearization ensures: local class first, then left-to-right bases,
# then parent classes. Base appears only once, after all subclasses.

print(D().greet())
# D → B → C → A → Base  — each super() hands off to next in MRO

# MRO inconsistency — this will raise TypeError at class definition
try:
    class X: pass
    class Y(X): pass
    class Z(X, Y): pass   # X before Y, but Y inherits X → inconsistent
except TypeError as e:
    print(f"Inconsistent MRO: {e}")

# Tracing greet() calls:
for cls in [Base, A, B, C, D]:
    obj = cls.__new__(cls)  # skip __init__ for demo
    # Only call greet if directly defined to show dispatch
    if "greet" in cls.__dict__:
        print(f"{cls.__name__}.greet defined locally")
```

---

## Exercise 18 — Mixins

```python
from datetime import datetime


class SerializableMixin:
    def to_dict(self) -> dict:
        return {k: v for k, v in self.__dict__.items() if not k.startswith("_")}

    @classmethod
    def from_dict(cls, d: dict):
        obj = cls.__new__(cls)
        obj.__dict__.update(d)
        return obj


class ValidatableMixin:
    def _validate(self):
        """Hook: subclasses raise ValueError if invalid."""
        pass

    def validate(self):
        self._validate()
        return True


class TimestampMixin:
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)  # cooperative — MUST call super()
        self.created_at = datetime.utcnow()
        self.updated_at = datetime.utcnow()

    def touch(self):
        self.updated_at = datetime.utcnow()


class User(TimestampMixin, ValidatableMixin, SerializableMixin):
    def __init__(self, name: str, email: str):
        super().__init__()   # triggers TimestampMixin.__init__ via MRO
        self.name = name
        self.email = email

    def _validate(self):
        if "@" not in self.email:
            raise ValueError(f"Invalid email: {self.email}")


u = User("Alice", "alice@example.com")
u.validate()
print(u.to_dict())
print(User.from_dict({"name": "Bob", "email": "bob@example.com"}).name)

# Mixin ordering matters:
# TimestampMixin must appear BEFORE classes that need timestamps in __init__.
# If ValidatableMixin came first and also had an __init__,
# TimestampMixin's __init__ might be skipped if super() isn't used.
# Rule: every mixin __init__ MUST call super().__init__(*args, **kwargs).
```

---

## Exercise 19 — Metaclasses

```python
# --- Part 1: runtime class creation with type() ---
MyDynClass = type("MyDynClass", (object,), {"greet": lambda self: "hello from dynamic class"})
obj = MyDynClass()
print(obj.greet())


# --- Part 2: auto-registering plugin metaclass ---
class PluginMeta(type):
    def __init__(cls, name, bases, namespace):
        super().__init__(name, bases, namespace)
        if not hasattr(cls, "_registry"):
            cls._registry = {}
        elif name != "Plugin":
            cls._registry[name] = cls


class Plugin(metaclass=PluginMeta):
    pass

class AudioPlugin(Plugin): pass
class VideoPlugin(Plugin): pass

print(Plugin._registry)
# {'AudioPlugin': <class 'AudioPlugin'>, 'VideoPlugin': <class 'VideoPlugin'>}


# --- Part 3: validation at class definition time ---
class ValidatedMeta(type):
    def __new__(mcs, name, bases, namespace):
        annotations = namespace.get("__annotations__", {})
        for field in annotations:
            if field[0].isdigit() or " " in field:
                raise TypeError(
                    f"Invalid field name {field!r} in class {name!r}"
                )
        return super().__new__(mcs, name, bases, namespace)


class GoodModel(metaclass=ValidatedMeta):
    name: str
    age: int

try:
    class BadModel(metaclass=ValidatedMeta):
        valid_field: str
        # "1bad" would cause TypeError at class definition
except TypeError as e:
    print(e)


# --- Part 4: __init_subclass__ alternative ---
class BasePlugin:
    _registry: dict = {}

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        BasePlugin._registry[cls.__name__] = cls

class ImagePlugin(BasePlugin): pass
class TextPlugin(BasePlugin): pass

print(BasePlugin._registry)
# __init_subclass__ is simpler for registration patterns;
# metaclasses are needed when you must intercept __new__ or __prepare__.
```

---

## Exercise 20 — Data Classes

```python
from dataclasses import dataclass, field, fields, asdict, replace
from datetime import datetime


@dataclass(frozen=True)
class Event:
    id: int
    name: str
    payload: dict = field(default_factory=dict, repr=False)
    timestamp: datetime = field(default_factory=datetime.utcnow)

    def __post_init__(self):
        if not self.name:
            raise ValueError("Event name cannot be empty")


e = Event(id=1, name="user.login", payload={"ip": "1.2.3.4"})
print(e)
# Event(id=1, name='user.login', timestamp=...) — payload hidden

try:
    e.id = 99   # FrozenInstanceError
except Exception as ex:
    print(ex)

# replace() creates a new instance with modified fields
e2 = replace(e, name="user.logout")
print(e2.name)

# asdict
print(asdict(e))

# fields() introspection
for f in fields(Event):
    print(f.name, f.type, f.repr)


@dataclass(order=True)
class Priority:
    level: int
    label: str

p1 = Priority(1, "low")
p2 = Priority(3, "high")
print(p1 < p2)    # True — comparison generated automatically
```

---

## Exercise 21 — `NamedTuple` & `TypedDict`

```python
from typing import NamedTuple, TypedDict, get_type_hints


class Coordinate(NamedTuple):
    lat: float
    lon: float
    altitude: float = 0.0


class GeoResponse(TypedDict):
    coordinates: Coordinate
    accuracy_meters: float
    source: str


# --- NamedTuple behaviors ---
c = Coordinate(37.7749, -122.4194, 15.0)

# Tuple unpacking
lat, lon, alt = c
print(lat, lon, alt)

# Index access
print(c[0])       # 37.7749

# _asdict()
print(c._asdict())   # {'lat': 37.7749, 'lon': -122.4194, 'altitude': 15.0}

# Immutability
try:
    c.lat = 0.0   # AttributeError
except AttributeError as e:
    print(e)

# --- TypedDict behaviors ---
response: GeoResponse = {
    "coordinates": c,
    "accuracy_meters": 5.0,
    "source": "GPS"
}

# TypedDict is TYPE HINT only — no runtime enforcement
print(isinstance(response, dict))          # True
print(isinstance(response, GeoResponse))   # TypeError — TypedDict is not a real class

# Introspect schema
print(get_type_hints(GeoResponse))

# When to choose:
# NamedTuple — when you need immutability, tuple protocol (unpacking/indexing),
#              and lightweight instances. Good for return values and small value objects.
# @dataclass  — when you need mutability, methods, inheritance, or frozen enforcement.
# TypedDict   — when you need to type-annotate existing dict-based APIs or JSON payloads
#               without changing the data structure (e.g., API response schemas).
```

---

## Bonus Challenge — `TinyORM` Solution

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from typing import Any
import uuid


# --- Descriptors ---
class Field:
    def __set_name__(self, owner, name):
        self.name = name
        self.private_name = f"_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self.private_name, None)

    def __set__(self, obj, value):
        self._validate(value)
        object.__setattr__(obj, self.private_name, value)

    def _validate(self, value):
        pass


class CharField(Field):
    def __init__(self, max_length: int = 255):
        self.max_length = max_length

    def _validate(self, value):
        if not isinstance(value, str):
            raise TypeError(f"{self.name} must be a str")
        if len(value) > self.max_length:
            raise ValueError(f"{self.name} max length is {self.max_length}")


class IntField(Field):
    def __init__(self, min_val: int = None, max_val: int = None):
        self.min_val = min_val
        self.max_val = max_val

    def _validate(self, value):
        if not isinstance(value, int):
            raise TypeError(f"{self.name} must be an int")
        if self.min_val is not None and value < self.min_val:
            raise ValueError(f"{self.name} must be >= {self.min_val}")
        if self.max_val is not None and value > self.max_val:
            raise ValueError(f"{self.name} must be <= {self.max_val}")


# --- Metaclass: collect fields ---
class ModelMeta(type):
    def __new__(mcs, name, bases, namespace):
        fields = {
            k: v for k, v in namespace.items()
            if isinstance(v, Field)
        }
        namespace["_fields"] = fields
        return super().__new__(mcs, name, bases, namespace)


# --- Abstract base ---
class Model(ABC, metaclass=ModelMeta):
    __slots__ = ("id",)

    @property
    @abstractmethod
    def table_name(self) -> str: ...

    @classmethod
    def create(cls, **kwargs) -> "Model":
        obj = cls.__new__(cls)
        object.__setattr__(obj, "id", str(uuid.uuid4())[:8])
        for key, val in kwargs.items():
            setattr(obj, key, val)
        return obj

    def __repr__(self):
        field_strs = ", ".join(
            f"{k}={getattr(self, k)!r}" for k in self._fields
        )
        return f"{type(self).__name__}(id={self.id!r}, {field_strs})"

    def __str__(self):
        return f"<{type(self).__name__} id={self.id}>"


# --- Mixin ---
class PersistenceMixin:
    _store: dict = {}

    def save(self):
        table = self.table_name
        if table not in PersistenceMixin._store:
            PersistenceMixin._store[table] = {}
        PersistenceMixin._store[table][self.id] = self
        return self

    @classmethod
    def load(cls, record_id: str) -> "Model":
        table = cls.table_name.fget(None) if isinstance(cls.table_name, property) else cls.table_name
        # Access via a temporary instance for table_name
        try:
            return PersistenceMixin._store[cls._get_table_name()][record_id]
        except KeyError:
            raise LookupError(f"No record with id={record_id!r}")

    @classmethod
    def _get_table_name(cls):
        # Instantiate minimally to get table_name
        return cls.__dict__.get("table_name", "unknown")


# --- Concrete model ---
class User(Model, PersistenceMixin):
    table_name = "users"
    name = CharField(max_length=100)
    age = IntField(min_val=0, max_val=150)

    __slots__ = ("_name", "_age")


# --- Demo ---
u = User.create(name="Alice", age=30)
print(repr(u))
print(str(u))

u.save()
print(f"Saved to store: {PersistenceMixin._store}")

# Validation
try:
    u.age = 200
except ValueError as e:
    print(f"Validation error: {e}")

try:
    u.name = "x" * 200
except ValueError as e:
    print(f"Validation error: {e}")
```
