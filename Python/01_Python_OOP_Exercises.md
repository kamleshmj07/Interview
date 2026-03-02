# Python OOP — Practical Exercises for Senior Developers

---

## CORE OOP CONCEPTS

---

### Exercise 1 — Classes & Objects/Instances

**Context:** You're building a trading system. Model a `StockPosition` that tracks a stock ticker, number of shares, and average cost basis.

**Tasks:**
1. Define the `StockPosition` class with appropriate instance attributes.
2. Create at least three distinct instances with different tickers.
3. Add an instance method `current_value(price: float) -> float` that returns the current market value.
4. Add an instance method `unrealized_pnl(price: float) -> float` that returns profit/loss vs. cost basis.
5. Demonstrate that mutating one instance does **not** affect others.

**Constraints:** No use of `@dataclass` or `NamedTuple` — implement manually.

---

### Exercise 2 — Attributes: Instance vs. Class

**Context:** You're designing a `DatabaseConnection` manager. Every connection shares the same DSN configuration (class-level), but each connection tracks its own query count (instance-level).

**Tasks:**
1. Define `_dsn` and `_pool_size` as **class attributes**.
2. Define `query_count` and `connection_id` as **instance attributes**.
3. Add a class method `configure(dsn, pool_size)` that updates the class attributes.
4. Show that updating a class attribute via the class itself propagates to all instances — but assigning via an instance shadows it.
5. Write a test that proves the shadowing behavior.

---

### Exercise 3 — Methods: Instance, Class & Static

**Context:** A `Temperature` utility class that supports multiple unit systems.

**Tasks:**
1. Implement instance methods `to_celsius()`, `to_fahrenheit()`, `to_kelvin()` based on the stored value and unit.
2. Implement `@classmethod` factory methods: `from_celsius(value)`, `from_fahrenheit(value)`, `from_kelvin(value)` — each returns a `Temperature` instance normalized to Celsius internally.
3. Implement a `@staticmethod` `is_valid_kelvin(value: float) -> bool` (must be ≥ 0).
4. Explain in a comment why `is_valid_kelvin` belongs as a static method rather than a class or instance method.

---

### Exercise 4 — `self` Parameter & `__init__` Constructor

**Context:** A `CircularBuffer` with a fixed capacity.

**Tasks:**
1. Implement `__init__(self, capacity: int)` — raise `ValueError` if capacity < 1.
2. Store internal state (`_buffer`, `_head`, `_tail`, `_size`) using `self`.
3. Implement `push(item)` — overwrites oldest when full.
4. Implement `pop() -> Any` — raises `IndexError` when empty.
5. Implement `__len__`.
6. Demonstrate why removing `self` from any method signature causes a `TypeError`, not a logic error.

---

### Exercise 5 — Single & Multiple Inheritance

**Context:** A notification system.

**Tasks:**
1. Define a base class `Notification` with `__init__(self, recipient, message)` and a method `send() -> str` that raises `NotImplementedError`.
2. Define `EmailNotification(Notification)` and `SMSNotification(Notification)` with concrete `send()` implementations.
3. Define `PriorityNotification` that inherits from **both** `EmailNotification` and `SMSNotification`, adds a `priority: int` attribute, and overrides `send()` to prepend `[PRIORITY-{n}]` to both channels.
4. Show the MRO of `PriorityNotification` and explain the diamond problem resolution.

---

### Exercise 6 — `super()`

**Context:** A multi-tier employee payroll system.

**Tasks:**
1. `Employee.__init__(name, base_salary)` — stores both attributes.
2. `Manager(Employee).__init__(name, base_salary, team_size)` — calls `super().__init__()`, adds `team_size`.
3. `Director(Manager).__init__(name, base_salary, team_size, budget)` — calls `super().__init__()`, adds `budget`.
4. Each class overrides `annual_compensation() -> float`:
   - `Employee`: `base_salary`
   - `Manager`: adds `team_size * 5000` bonus
   - `Director`: adds `budget * 0.01` bonus on top of `Manager`'s calculation using `super()`
5. Demonstrate that removing any `super()` call silently breaks the bonus chain.

---

### Exercise 7 — Encapsulation (`_single`, `__name_mangling`)

**Context:** A `BankAccount` class.

**Tasks:**
1. Use `__balance` (private, name-mangled) for the balance.
2. Use `_transaction_log` (protected, convention-only) for a list of transaction strings.
3. Implement `deposit(amount)` and `withdraw(amount)` — raise `ValueError` for invalid amounts; `withdraw` raises `InsufficientFundsError` (custom exception) when balance is too low.
4. Show that `account.__balance` raises `AttributeError` but `account._BankAccount__balance` works — and explain why this is a feature, not a bug.
5. Demonstrate that `_transaction_log` is accessible but signals "internal use".

---

### Exercise 8 — Polymorphism: Duck Typing & Method Overriding

**Context:** A document export pipeline.

**Tasks:**
1. Define three unrelated classes (no shared base): `PDFExporter`, `CSVExporter`, `JSONExporter` — each with `export(data: dict) -> str`.
2. Write a function `run_export(exporter, data)` that calls `exporter.export(data)` without any `isinstance` checks.
3. Demonstrate duck typing by passing all three exporters through `run_export`.
4. Now add a base class `BaseExporter` with `export` as an overridable method, and refactor your exporters.
5. Override `export` in a `CompressedCSVExporter(CSVExporter)` that wraps the CSV output in a mock compression step.
6. Show how method overriding differs from duck typing in terms of coupling.

---

### Exercise 9 — Abstraction: Abstract Base Classes (`abc`)

**Context:** A plugin system for data validators.

**Tasks:**
1. Define an ABC `DataValidator` with:
   - Abstract method `validate(value) -> bool`
   - Abstract method `error_message() -> str`
   - Concrete method `check(value) -> str` that calls both and returns a formatted result.
2. Implement `EmailValidator`, `PhoneValidator`, and `AgeValidator`.
3. Show that instantiating `DataValidator` directly raises `TypeError`.
4. Register a third-party class (without modifying it) as a virtual subclass using `DataValidator.register(...)`, and show that `isinstance` works but the abstract methods are **not** enforced.
5. Explain when you'd choose virtual subclassing over inheritance.

---

## ADVANCED OOP CONCEPTS

---

### Exercise 10 — Properties (`@property`, setter, deleter)

**Context:** A `Rectangle` class where `area` and `perimeter` are computed, and width/height are validated.

**Tasks:**
1. Use `@property` with `@<name>.setter` for `width` and `height` — reject non-positive values.
2. Expose `area` and `perimeter` as read-only computed properties (no setter).
3. Add a `diagonal` property.
4. Add a `scale` property with a setter that scales both dimensions proportionally.
5. Implement a `@<name>.deleter` for `width` that resets it to 1.0.
6. Demonstrate that assigning to `rect.area` raises `AttributeError`.

---

### Exercise 11 — `@classmethod` Deep Dive

**Context:** A `Config` class that should support multiple serialization formats.

**Tasks:**
1. Implement `Config(data: dict)` storing `_data`.
2. Add `@classmethod from_json(cls, json_str)` and `@classmethod from_env(cls, prefix)` (reads from `os.environ` keys starting with prefix).
3. Add `@classmethod default(cls)` that returns a Config with sensible defaults.
4. Subclass `AppConfig(Config)` and override `default()` with app-specific defaults.
5. Show that calling `AppConfig.default()` returns an `AppConfig` instance, not a `Config` instance — and explain why `cls` instead of the hard-coded class name matters.

---

### Exercise 12 — `@staticmethod` Deep Dive

**Context:** A `MathUtils` helper namespace.

**Tasks:**
1. Implement these as `@staticmethod`: `clamp(value, min_val, max_val)`, `lerp(a, b, t)`, `is_prime(n)`, `fibonacci(n)`.
2. Show they can be called on both the class and an instance.
3. Refactor `is_prime` into a module-level function, then debate in a comment block: when should logic live as a static method vs. a module-level function?
4. Show that `@staticmethod` cannot access `cls` or `self` — and why that's intentional.

---

### Exercise 13 — `__str__` and `__repr__`

**Context:** A `Vector3D` class.

**Tasks:**
1. Implement `__repr__` to return `Vector3D(x=1.0, y=2.0, z=3.0)` — must be `eval()`-able.
2. Implement `__str__` to return `<1.0, 2.0, 3.0>`.
3. Demonstrate the difference: `repr(v)`, `str(v)`, `f"{v}"`, `f"{v!r}"`, and what `[v]` shows in a list.
4. Add `__format__` so `f"{v:.2f}"` rounds each component.
5. Explain why a class with only `__repr__` still works in `str()` contexts.

---

### Exercise 14 — `__slots__` for Memory Optimization

**Context:** A high-frequency `Tick` data class storing symbol, price, volume, and timestamp for millions of records.

**Tasks:**
1. Implement `Tick` with `__slots__`.
2. Show that adding an undeclared attribute raises `AttributeError`.
3. Use the `tracemalloc` module to compare memory usage between 100,000 `Tick` instances and 100,000 equivalent plain-dict-backed objects.
4. Subclass `Tick` as `ExtendedTick` that adds a `vwap` slot — show you must declare slots in the subclass too.
5. Show that `__slots__` classes cannot have a `__dict__` by default, and how to opt back in.

---

### Exercise 15 — `__getattr__` / `__setattr__` / `__delattr__`

**Context:** A `DynamicConfig` object that proxies attribute access to an internal dictionary and supports dot-notation access with audit logging.

**Tasks:**
1. Implement `__getattr__` to look up missing attributes in `self._store`.
2. Implement `__setattr__` to intercept all attribute sets — store `_store` and `_log` via `object.__setattr__`, and everything else in `_store` with a log entry.
3. Implement `__delattr__` to remove from `_store` and log the deletion.
4. Implement `__dir__` to merge `_store` keys with normal attributes (useful for tab-completion).
5. Show the infinite recursion risk in `__setattr__` when calling `self.x = y` directly, and why `object.__setattr__` is the fix.

---

### Exercise 16 — `__call__` (Callable Objects)

**Context:** A configurable `RateLimiter` that can be used as both a decorator and a direct callable guard.

**Tasks:**
1. Implement `RateLimiter(max_calls: int, period: float)` with `__call__(self, func)` that wraps `func` to enforce rate limiting.
2. Use it as `@RateLimiter(5, 1.0)` on a function.
3. Also implement a standalone usage: `limiter = RateLimiter(10, 1.0); limiter.check()` raises `RateLimitExceeded` or returns `True`.
4. Show `callable(limiter)` returns `True`.
5. Add `__repr__` showing current call count and limit.

---

### Exercise 17 — Method Resolution Order (MRO)

**Context:** A mixin-heavy logging/caching framework.

**Tasks:**
1. Construct a diamond hierarchy: `Base → A, Base → B → A` (classic diamond).
2. Print and explain the MRO for each class using `ClassName.__mro__` and `mro()`.
3. Build a case where the MRO is **inconsistent** (C3 linearization violation) and show the `TypeError` Python raises.
4. Add a `greet()` method at multiple levels and trace exactly which implementation runs and why for each class.
5. Explain when relying on implicit MRO is acceptable vs. when you should call `super()` explicitly with type arguments.

---

### Exercise 18 — Mixins

**Context:** A set of reusable capability mixins for model classes.

**Tasks:**
1. Implement `SerializableMixin` with `to_dict()` and `from_dict(cls, d)` (classmethod).
2. Implement `ValidatableMixin` with an abstract-ish `_validate()` hook and a `validate()` method that calls it.
3. Implement `TimestampMixin` that auto-sets `created_at` and `updated_at` on `__init__` and save.
4. Compose them: `class User(TimestampMixin, ValidatableMixin, SerializableMixin): ...`.
5. Show mixin ordering matters — swap `TimestampMixin` and `ValidatableMixin` and observe the MRO difference.
6. Enforce a convention: mixins should **never** define `__init__` without calling `super().__init__()` — demonstrate what breaks when they don't.

---

### Exercise 19 — Metaclasses (`type` and Custom Metaclasses)

**Context:** A self-registering plugin system and an ORM-style field declaration system.

**Tasks:**
1. Use `type(name, bases, namespace)` to programmatically create a class at runtime.
2. Implement a `PluginMeta` metaclass that auto-registers every subclass of `Plugin` into a `Plugin._registry` dict keyed by class name.
3. Implement a `ValidatedMeta` metaclass that inspects class-body annotations and raises `TypeError` at **class definition time** if any annotated field name starts with a digit or contains spaces.
4. Show the order: `__prepare__` → `__new__` → `__init__` → instance creation.
5. Convert one of the above to use `__init_subclass__` instead and compare the approaches.

---

### Exercise 20 — Data Classes (`@dataclass`)

**Context:** A type-safe event system.

**Tasks:**
1. Define `@dataclass class Event` with fields: `id: int`, `name: str`, `payload: dict`, `timestamp: datetime` (default `field(default_factory=datetime.utcnow)`).
2. Make it **frozen** (`frozen=True`) and show that mutation raises `FrozenInstanceError`.
3. Add `__post_init__` to validate that `name` is non-empty.
4. Use `field(repr=False)` on `payload` to hide it from `repr`.
5. Define `@dataclass(order=True) class Priority` and show that comparison operators work automatically.
6. Demonstrate `dataclasses.asdict`, `dataclasses.replace`, and `dataclasses.fields`.

---

### Exercise 21 — `NamedTuple` & `TypedDict`

**Context:** A geolocation service returning structured coordinate data.

**Tasks:**
1. Define a `Coordinate` using `typing.NamedTuple` with fields `lat: float`, `lon: float`, `altitude: float = 0.0`.
2. Show that `Coordinate` supports tuple unpacking, indexing, and `_asdict()`.
3. Show that `Coordinate` is **immutable** — mutation raises `AttributeError`.
4. Define a `GeoResponse` using `TypedDict` with `coordinates: Coordinate`, `accuracy_meters: float`, `source: str`.
5. Demonstrate that `TypedDict` is a **type-hint tool**, not a runtime enforcer — show that a plain dict passes `isinstance(d, dict)` but not `isinstance(d, GeoResponse)`.
6. Use `typing.get_type_hints(GeoResponse)` to introspect the schema at runtime.
7. Discuss: when would you choose `NamedTuple` over `@dataclass`? When `TypedDict`?

---

## Bonus Challenge — Bring It All Together

Design a mini **ORM-style framework** called `TinyORM` that uses:

- A **metaclass** to auto-collect field descriptors from class body
- **Descriptors** (`__get__`/`__set__`) for typed field access with validation
- A **`@classmethod`** `create(**kwargs)` factory
- **`__slots__`** for memory efficiency
- **`__repr__`** and **`__str__`** for clean output
- A **mixin** `PersistenceMixin` with `save()` / `load(id)` backed by an in-memory dict store
- An **abstract base class** `Model` enforcing a `table_name` class attribute

You should be able to write:

```python
class User(Model, PersistenceMixin):
    table_name = "users"
    name = CharField(max_length=100)
    age = IntField(min_val=0, max_val=150)

u = User.create(name="Alice", age=30)
u.save()
loaded = User.load(u.id)
```
