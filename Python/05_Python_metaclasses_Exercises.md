# Python Metaclasses — Practical Exercises for Senior Developers

> **Added topics** (beyond your original list):
> * `__prepare__` — namespace customization before class body executes
> * `__call__` on metaclasses — controlling instance creation
> * `__instancecheck__` / `__subclasscheck__` — customizing `isinstance`/`issubclass`
> * `__init_subclass__` — lightweight metaclass alternative on the base class
> * Cooperative metaclasses — using `super()` inside metaclass methods
> * `type.__new__` vs `object.__new__` — where class vs instance creation diverges
> * Metaclasses and descriptors — combining both for ORM-style field validation
> * Thread-safe Singleton — extending the basic singleton pattern

---

## FOUNDATIONS

---

### Exercise 1 — `type`: The Default Metaclass

**Context:** Before writing custom metaclasses, you need to master `type` as both an introspection tool and a dynamic class factory — skills used in plugin loaders and code-generation tools.

**Tasks:**
1. Confirm the metaclass chain: `type(42)`, `type(int)`, `type(type)` — explain what it means that `type(type) is type`.
2. Use the three-argument `type(name, bases, namespace)` form to dynamically create:
   - A `Point` class with `x` and `y` as class attributes and a `distance()` method.
   - A `ColorPoint(Point)` subclass that adds a `color` attribute.
   - Verify `isinstance(ColorPoint(), Point)` is `True`.
3. Create a class using `type(...)` that has a `__repr__`, a `classmethod`, and a `staticmethod` — all defined in the namespace dict.
4. Show that `MyClass.__class__` is `type`, and `type.__class__` is also `type` — explain the bootstrapping.
5. Use `type.__mro__` to verify `type` inherits from `object`, while `object`'s metaclass is `type`.
6. Demonstrate that `type(name, bases, namespace)` is exactly what Python calls behind the scenes for a `class` statement — prove by comparing `id` behaviour and attribute identity.

**Constraint:** No `class` keyword for the dynamically created classes.

---

### Exercise 2 — `__new__` vs `__init__` in Metaclasses

**Context:** A code-generation framework where class creation must be intercepted at two distinct phases — before the class object exists (`__new__`) and after it is created (`__init__`).

**Tasks:**
1. Implement `TracingMeta(type)` that overrides both `__new__` and `__init__`. Print which phase is running and what arguments are available at each stage.
2. Show the exact call order: `TracingMeta.__new__` → `TracingMeta.__init__` → instance `__new__` → instance `__init__` by creating a class and then instantiating it, with print statements in all four methods.
3. In `__new__`, inject an additional class attribute `_created_at` (using `datetime.utcnow()`) into the namespace **before** the class object is created — show it is present on the finished class.
4. In `__init__`, register the class in a module-level `CLASS_REGISTRY` dict — show that `__new__` can't do this cleanly (the class doesn't exist yet as a final object).
5. Show the difference: modifying `namespace` in `__new__` vs calling `setattr(cls, ...)` in `__init__` — when does each approach fail?
6. Demonstrate that returning a completely **different** object from `__new__` (not the result of `super().__new__(...)`) causes `__init__` to be skipped if the returned object's type differs.

---

### Exercise 3 — `__prepare__`: Namespace Customization

**Context:** A DSL for defining database schemas where column definition order must be preserved and duplicate column names must be rejected at class-definition time.

**Tasks:**
1. Implement `SchemaMeta(type)` with `__prepare__` returning an `OrderedDict`. Show that without `__prepare__`, Python 3.7+ dicts preserve insertion order anyway — explain when `__prepare__` still matters.
2. Implement `StrictNamespaceMeta(type)` — `__prepare__` returns a custom `dict` subclass `StrictNamespace` that raises `DuplicateAttributeError` when the same name is assigned twice in the class body.
3. Show that `__prepare__` receives `name` and `bases` as arguments — use `bases` to inherit namespace rules from parent schemas.
4. Implement `AuditNamespaceMeta(type)` — `__prepare__` returns a namespace that records the **order** each attribute was defined, storing it as `cls._definition_order` on the finished class.
5. Show that `__prepare__` is called **before** the class body executes — prove by printing inside `__prepare__` and inside the class body.
6. Demonstrate that `__prepare__` is a `classmethod` by convention — call `super().__prepare__(name, bases, **kwargs)` and show what the base `type.__prepare__` returns.

---

## CORE PATTERNS

---

### Exercise 4 — `metaclass=` Keyword & Metaclass Inheritance

**Context:** A multi-tier plugin framework where metaclass behaviour must be inherited without re-specifying it on every subclass.

**Tasks:**
1. Define `PluginMeta(type)` and apply it with `class BasePlugin(metaclass=PluginMeta)`. Show that subclasses `AudioPlugin(BasePlugin)` and `VideoPlugin(BasePlugin)` automatically use `PluginMeta` — verify with `type(AudioPlugin) is PluginMeta`.
2. Show that the `metaclass=` keyword is **only** needed on the root class — subclasses inherit the metaclass transparently.
3. Implement a two-level metaclass hierarchy: `BaseMeta(type)` → `ExtendedMeta(BaseMeta)`. Apply `ExtendedMeta` to a class and show both `BaseMeta.__new__` and `ExtendedMeta.__new__` run via `super()`.
4. Demonstrate **metaclass conflict**: create `MetaA(type)` and `MetaB(type)`, then try `class C(A, B)` where `A` uses `MetaA` and `B` uses `MetaB`. Show the `TypeError`. Fix it by creating `MetaC(MetaA, MetaB)` as the resolution metaclass.
5. Show that when Python resolves the metaclass for a class with multiple bases, it picks the **most derived** metaclass among all bases' metaclasses — document the algorithm in comments.
6. Implement `__init_subclass__` on `BasePlugin` as a simpler alternative to metaclass subclass tracking. Compare the two approaches side by side — list when each is preferable.

---

### Exercise 5 — MRO Customization via Metaclasses

**Context:** A framework that needs to alter or validate the Method Resolution Order before a class is finalised.

**Tasks:**
1. In a metaclass `__new__`, call `mro_result = super().__new__(mcs, name, bases, namespace).mro()` and print the MRO for every class created — a transparent MRO tracer.
2. Implement `StrictMROMeta(type)` — raises `TypeError` during class creation if any class in the MRO (other than `object`) appears more than once in the bases tuple.
3. Implement `MROValidatorMeta(type)` — raises `TypeError` if any method defined in the class body **shadows** a method from a parent with a different signature (use `inspect.signature` to compare).
4. Override `mro()` method on a metaclass to **inject** a mixin class into the MRO of every class created with that metaclass — without the mixin appearing in `__bases__`. Show the MRO before and after.
5. Demonstrate that `mro()` on the metaclass is what Python calls to build `__mro__` — not directly accessible to the class itself.
6. Show a case where a custom `mro()` breaks `super()` chaining and explain why — then fix it.

---

### Exercise 6 — Abstract Base Classes via Metaclasses

**Context:** Replicating Python's `abc.ABCMeta` behaviour from scratch to understand its internals — then using the real `ABCMeta` in a plugin validation system.

**Tasks:**
1. Implement `SimpleABCMeta(type)` — tracks methods decorated with a custom `@abstract` marker in `__new__`, stores them in `cls.__abstract_methods__`. Override `__call__` to raise `TypeError` if any abstract methods remain unimplemented at instantiation time.
2. Show that `ABCMeta.__instancecheck__` makes `isinstance` work for virtual subclasses — implement the same check in `SimpleABCMeta`.
3. Use the real `abc.ABCMeta` (or `ABC` base class) to define a `Serializable` ABC with abstract methods `serialize() -> bytes` and `deserialize(data: bytes)`. Implement two concrete classes. Show `TypeError` on direct instantiation.
4. Register a third-party class as a **virtual subclass** using `Serializable.register(ThirdParty)` — show `isinstance` passes but abstract methods are not enforced.
5. Implement `@abstractproperty` (deprecated in stdlib — re-implement it) combining `@property` and `@abstract` via a descriptor that sets an `__isabstractmethod__` flag.
6. Show the interaction: a class that implements all abstract methods **except one inherited from a grandparent** is still abstract — trace the `__abstractmethods__` frozenset at each level.

---

### Exercise 7 — Singleton Enforcement

**Context:** A configuration manager, a connection pool, and a logger — each must guarantee exactly one instance exists globally, including across subclasses, threads, and module re-imports.

**Tasks:**
1. Implement `SingletonMeta(type)` — stores one instance per class in `_instances: dict`. Override `__call__` to return the cached instance on subsequent calls.
2. Show that subclassing a Singleton class creates a **separate** singleton per subclass — `Config` and `AppConfig(Config)` are different singletons.
3. Show the **thread-safety problem**: simulate two threads calling `instance()` simultaneously without a lock — show the race condition (use `threading` and a sleep inside `__call__` before the instance is stored).
4. Fix with `threading.Lock` per class — implement `ThreadSafeSingletonMeta`.
5. Implement a `ResetableSingletonMeta` — adds `cls._reset_instance()` classmethod to clear the cached instance (useful for testing).
6. Compare `SingletonMeta` with the module-level singleton pattern (just a module with state) and the Borg pattern (shared `__dict__`) — implement all three and discuss trade-offs.
7. Show that `pickle` breaks naive singletons — `pickle.loads(pickle.dumps(s)) is s` is `False`. Implement `__reduce__` on the singleton class to fix it.

---

## ADVANCED PATTERNS

---

### Exercise 8 — Attribute Validation & Registration at Class Creation

**Context:** An ORM-style field system where field declarations in the class body are validated and registered automatically when the class is defined — not when instances are created.

**Tasks:**
1. Implement `Field` base descriptor and subclasses `CharField(max_length)`, `IntField(min_val, max_val)`, `FloatField()`. Each has `__set_name__` to capture the attribute name.
2. Implement `ModelMeta(type)` — in `__new__`, scan the namespace for `Field` instances, collect them into `cls._fields: dict`, and raise `TypeError` if:
   - A field name starts with an underscore.
   - Two fields have the same database column name (via a `column=` kwarg).
3. Implement `ModelMeta.__init__` to auto-generate `__init__`, `__repr__`, and `__eq__` on the class based on `_fields`.
4. Add **index registration**: a field decorated with `Field(index=True)` should be auto-added to `cls._indexes: list` at class creation.
5. Implement `ModelMeta.__call__` to validate all field values against their descriptors before the instance is returned — raise `ValidationError` with a summary of all failures, not just the first.
6. Show the complete working model:
   ```python
   class User(Model):
       name = CharField(max_length=100)
       age  = IntField(min_val=0, max_val=150)
       score = FloatField()
   ```
   Creating `User(name="Alice", age=30, score=9.5)` should work; `User(name="x"*200, age=-1)` should raise `ValidationError` listing both failures.

---

### Exercise 9 — Dynamic Method Addition & Modification

**Context:** A test framework that auto-generates test runner methods, and an API client that generates endpoint methods from a spec dict — all via metaclass-level code generation.

**Tasks:**
1. Implement `TestSuiteMeta(type)` — scans the class body for methods starting with `test_`, wraps each with a timer and exception catcher, and adds `run_all()` which executes every test and prints a summary report.
2. Implement `APIMeta(type)` — reads a class-level `ENDPOINTS: dict` of `{"method_name": ("HTTP_VERB", "/path")}` and dynamically generates methods on the class that simulate HTTP calls (return a dict with verb, path, and args).
3. Implement `LoggingMeta(type)` — wraps every non-dunder method in the class body with a logging wrapper that prints `[CALL] ClassName.method_name(args)` on entry and `[RETURN] value` on exit. Use `functools.wraps`.
4. In `__new__`, show how to add methods that correctly reference `cls` (the class being created) — demonstrate the closure-capture problem and fix it with a factory function.
5. Implement `DeprecationMeta(type)` — reads a `_deprecated: dict` mapping old method names to new ones, auto-generates shim methods that call the new method with a `DeprecationWarning`.
6. Show that modifying `namespace` in `__new__` is safer than calling `setattr` in `__init__` for injected methods — explain why (hint: descriptors, `__set_name__`).

---

### Exercise 10 — `__instancecheck__` / `__subclasscheck__`

**Context:** A type system for a data-science library where "duck-type" compatibility checks must work with `isinstance` and `issubclass` without requiring formal inheritance.

**Tasks:**
1. Implement `ProtocolMeta(type)` — override `__instancecheck__(cls, instance)` to check whether `instance` has all methods listed in `cls._required_methods`, returning `True` even if there is no inheritance relationship.
2. Implement `__subclasscheck__(cls, subclass)` to return `True` if `subclass` defines all required methods — making `issubclass(Duck, Swimmable)` work structurally.
3. Create `Swimmable`, `Flyable`, and `Runnable` protocols using `ProtocolMeta`. Show that a `Duck` class with `swim()` and `fly()` passes `isinstance(duck, Swimmable)` and `isinstance(duck, Flyable)` but not `isinstance(duck, Runnable)`.
4. Show the interaction with `try/except TypeError` — `issubclass` raises `TypeError` for non-class first arguments; show your `__subclasscheck__` handles this correctly.
5. Implement `CompositeProtocol` — combines two protocols: `isinstance(obj, And(Swimmable, Flyable))` returns `True` only if both checks pass. Use `__instancecheck__` on a metaclass-powered `And` class.
6. Explain and demonstrate why overriding `__instancecheck__` on a regular class (not the metaclass) does **not** affect `isinstance` — the dunder must live on the **metaclass**, not the class.

---

### Exercise 11 — `__call__` on Metaclasses: Controlling Instance Creation

**Context:** An object pool, a flyweight factory, and a type-coercion system — all requiring control over what `ClassName(...)` actually returns.

**Tasks:**
1. Show the full `__call__` chain: `MyMeta.__call__` → `MyClass.__new__` → `MyClass.__init__`. Override all three with print statements to prove the order and which can abort the chain.
2. Implement `PoolMeta(type)` — `__call__` maintains a pool of pre-created instances per class. Returns an idle instance from the pool if available; creates a new one only when the pool is empty. Adds a `release()` method to return instances to the pool.
3. Implement `FlyweightMeta(type)` — `__call__` interns instances by their constructor arguments. `Point(1, 2)` called twice returns the **same object** (`is` check). Use a `WeakValueDictionary` so unused instances can be garbage collected.
4. Implement `CoercingMeta(type)` — `__call__` inspects the first argument's type; if it already is an instance of the class, return it unchanged (no double-wrapping). Useful for `MyType(x)` being safe to call on already-converted values.
5. Show that returning `None` or a non-instance from `__call__` is legal — build a `NullableMeta` where `MyClass(None)` returns `None` rather than an instance.
6. Demonstrate that overriding `__call__` on the **metaclass** affects class instantiation, while overriding `__call__` on the **class** only affects calling instances of that class — show both side by side.

---

### Exercise 12 — `__init_subclass__` vs Metaclasses

**Context:** Choosing the right tool — `__init_subclass__` covers most subclass-registration use cases with far less complexity than a full metaclass.

**Tasks:**
1. Implement the same plugin registry pattern two ways:
   - `PluginMeta(type)` — uses `__new__` to detect subclasses and register them.
   - `BasePlugin` with `__init_subclass__` — no metaclass needed.
   Show that both produce identical registries, but `__init_subclass__` is ~70% less code.
2. Show that `__init_subclass__` receives `**kwargs` passed as class keywords: `class MyPlugin(BasePlugin, version="2.0")` — the `version` kwarg is forwarded to `__init_subclass__`.
3. Show the **limits** of `__init_subclass__`:
   - Cannot modify the namespace before the class body executes (`__prepare__` is unavailable).
   - Cannot intercept instance creation (`__call__` on metaclass is unavailable).
   - Cannot override MRO.
4. Implement `class Validated` using `__init_subclass__` to enforce that every subclass defines a `schema: dict` class attribute — raise `TypeError` at subclass definition time if missing.
5. Show `super().__init_subclass__(**kwargs)` is mandatory for cooperative subclassing — demonstrate what breaks in a diamond hierarchy when it's missing.
6. Build a decision matrix in comments: for each of the 12 metaclass use cases in this exercise set, state whether `__init_subclass__` can replace the metaclass and why.

---

### Exercise 13 — Cooperative Metaclasses & `super()` Inside Metaclasses

**Context:** A framework combining third-party libraries — each brings its own metaclass. Making them coexist requires cooperative `super()` usage throughout.

**Tasks:**
1. Show the problem: `class C(A, B)` where `A` uses `MetaA` and `B` uses `MetaB` raises `TypeError: metaclass conflict`. Reproduce the error cleanly.
2. Implement `CoopMeta(MetaA, MetaB)` as the resolution. Show that both `MetaA.__new__` and `MetaB.__new__` are called via `super()` — insert print statements to prove it.
3. Show that `super()` in a metaclass `__new__` calls the next metaclass in **the metaclass's own MRO**, not the class's MRO — print `CoopMeta.__mro__` to confirm.
4. Implement three metaclasses each adding a different class attribute (`_meta_a`, `_meta_b`, `_meta_c`) via `super().__new__` in a chain. Show the composed class has all three.
5. Show the fragile case: a metaclass that does `type.__new__(mcs, ...)` directly (bypassing `super()`) breaks cooperation — demonstrate and fix.
6. Implement a `mixin_meta(*metaclasses)` factory function that dynamically creates a cooperative merged metaclass from any list of metaclasses — handles the conflict resolution automatically.

---

### Exercise 14 — Class Decorators vs Metaclasses

**Context:** The same five transformations implemented both ways — comparing when each tool is the right choice.

**Tasks:**
Implement the following **five transformations** using both a class decorator and a metaclass. After each pair, write a comparison comment:

1. **Auto-repr**: inject `__repr__` based on `__init__` signature.
2. **Registry**: auto-register every subclass in a central dict.
3. **Frozen**: prevent attribute mutation after `__init__`.
4. **Method logging**: wrap all non-dunder methods with call logging.
5. **Field validation**: scan class body for descriptor fields and validate them at definition time.

For each pair:
- Show the class decorator version first, then the metaclass version.
- Note which is simpler, which is more powerful, and one scenario where the decorator approach **fails** and the metaclass is required.

**Final task:** Write a `@class_decorator_to_meta(decorator_fn)` factory that converts any class decorator into an equivalent metaclass — show it working on your `auto_repr` decorator.

---

## Bonus Challenge — Bring It All Together

Design **`ModelForge`** — a mini ORM metaclass system that combines every concept from this module.

**Requirements:**

- `ModelMeta` inherits from both `ABCMeta` and a custom `ValidatingMeta` cooperatively.
- `__prepare__` returns a `FieldOrderedNamespace` that rejects duplicate field names.
- `__new__` collects `Field` descriptors, validates names, checks for `table_name` abstract attribute, injects `__repr__`/`__eq__`/`__hash__`, and registers the model in `ModelForge.registry`.
- `__init__` registers indexes, sets up a `_hooks: dict` for pre/post save hooks.
- `__call__` validates all field values before returning the instance (all errors at once).
- `__instancecheck__` on the metaclass makes `isinstance(obj, Model)` return `True` for any object that has all required fields, even without inheritance.
- `SingletonMeta` applied to `ModelForge` itself (the registry manager) — only one registry exists.
- `__init_subclass__` on `Model` handles optional `abstract = True` class keyword to skip registration.
- Thread-safe instance caching via `__call__` override for read-only models.

You should be able to write:

```python
class User(Model, abstract=False):
    table_name = "users"
    name  = CharField(max_length=100, index=True)
    age   = IntField(min_val=0, max_val=150)
    email = CharField(max_length=200, column="email_address")

u = User(name="Alice", age=30, email="alice@example.com")
print(repr(u))                          # User(name='Alice', age=30, email='alice@...')
print(ModelForge.registry)              # {'User': <class 'User'>}
print(isinstance(u, Model))            # True (structural check)
User(name="x"*200, age=-1, email="")   # ValidationError listing all 3 failures
```
