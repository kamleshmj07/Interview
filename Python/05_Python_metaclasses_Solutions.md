# Python Metaclasses — Solutions

---

## Exercise 1 — `type`: The Default Metaclass

```python
import math

print(type(42))       # <class 'int'>
print(type(int))      # <class 'type'>
print(type(type))     # <class 'type'>  ← type is its own metaclass (self-bootstrapping)
# type(type) is type because type is the root metaclass — it created itself.

# Dynamic class creation — no class keyword
def _distance(self):
    return math.hypot(self.x, self.y)

Point = type("Point", (object,), {
    "x": 0.0,
    "y": 0.0,
    "distance": _distance,
    "__repr__": lambda self: f"Point({self.x}, {self.y})",
})

ColorPoint = type("ColorPoint", (Point,), {
    "color": "black",
    "__repr__": lambda self: f"ColorPoint({self.x}, {self.y}, {self.color})",
})

cp = ColorPoint()
cp.x, cp.y, cp.color = 3.0, 4.0, "red"
print(cp, cp.distance())          # ColorPoint(3.0, 4.0, red) 5.0
print(isinstance(cp, Point))      # True

# classmethod and staticmethod in namespace dict
MyClass = type("MyClass", (object,), {
    "__repr__": lambda self: "MyClass()",
    "greet": classmethod(lambda cls: f"Hello from {cls.__name__}"),
    "add":   staticmethod(lambda a, b: a + b),
})
print(MyClass.greet())    # Hello from MyClass
print(MyClass.add(1, 2))  # 3

# Metaclass bootstrapping
print(MyClass.__class__ is type)  # True
print(type.__class__ is type)     # True — type is its own metaclass

print(issubclass(type, object))   # True — type inherits object
print(type(object) is type)       # True — object's metaclass is type
```

---

## Exercise 2 — `__new__` vs `__init__` in Metaclasses

```python
from datetime import datetime, UTC

CLASS_REGISTRY = {}

class TracingMeta(type):
    def __new__(mcs, name, bases, namespace):
        # Phase 1: class object does not yet exist
        # Safe to modify namespace before the class is built
        print(f"  __new__: creating '{name}'")
        namespace["_created_at"] = datetime.now(UTC)
        cls = super().__new__(mcs, name, bases, namespace)
        return cls

    def __init__(cls, name, bases, namespace):
        # Phase 2: cls is now a fully formed class object
        print(f"  __init__: finalising '{name}'")
        CLASS_REGISTRY[name] = cls   # registering here is clean and safe
        super().__init__(name, bases, namespace)


class MyTracedClass(metaclass=TracingMeta):
    def __new__(cls):
        print("    instance __new__")
        return super().__new__(cls)

    def __init__(self):
        print("    instance __init__")

# Call order: TracingMeta.__new__ → TracingMeta.__init__ → MyTracedClass.__new__ → MyTracedClass.__init__
obj = MyTracedClass()
print(MyTracedClass._created_at)   # datetime injected in __new__
print(CLASS_REGISTRY)

# Returning a non-type from __new__ skips __init__
class SkippingMeta(type):
    def __new__(mcs, name, bases, namespace):
        if name == "SpecialClass":
            return 42   # not a type — __init__ is skipped entirely
        return super().__new__(mcs, name, bases, namespace)

    def __init__(cls, name, bases, namespace):
        print(f"  SkippingMeta.__init__ for {name}")
        if isinstance(cls, type):
            super().__init__(name, bases, namespace)

result = SkippingMeta("SpecialClass", (), {})
print(result)   # 42 — __init__ was NOT called
```

---

## Exercise 3 — `__prepare__`: Namespace Customization

```python
from collections import OrderedDict


class DuplicateAttributeError(Exception): pass


class StrictNamespace(dict):
    """Raises on duplicate non-dunder assignments in the class body."""
    def __setitem__(self, key, value):
        if key in self and not key.startswith("__"):
            raise DuplicateAttributeError(f"Duplicate attribute: {key!r}")
        super().__setitem__(key, value)


class StrictNamespaceMeta(type):
    @classmethod
    def __prepare__(mcs, name, bases, **kwargs):
        print(f"  __prepare__ called for '{name}' BEFORE class body executes")
        return StrictNamespace()   # class body writes into this

    def __new__(mcs, name, bases, namespace, **kwargs):
        return super().__new__(mcs, name, bases, dict(namespace))


class GoodSchema(metaclass=StrictNamespaceMeta):
    x = 1
    y = 2

try:
    class BadSchema(metaclass=StrictNamespaceMeta):
        x = 1
        x = 2   # DuplicateAttributeError at class-definition time
except DuplicateAttributeError as e:
    print(f"Caught at class creation: {e}")


# AuditNamespaceMeta — records definition order
class AuditNamespace(dict):
    def __init__(self):
        super().__init__()
        self._order = []

    def __setitem__(self, key, value):
        if not key.startswith("__"):
            self._order.append(key)
        super().__setitem__(key, value)


class AuditNamespaceMeta(type):
    @classmethod
    def __prepare__(mcs, name, bases, **kwargs):
        return AuditNamespace()

    def __new__(mcs, name, bases, namespace, **kwargs):
        order = namespace._order[:]
        cls = super().__new__(mcs, name, bases, dict(namespace))
        cls._definition_order = order
        return cls


class MyAudited(metaclass=AuditNamespaceMeta):
    z = 10
    a = 20
    b = 30

print(MyAudited._definition_order)   # ['z', 'a', 'b']

# __prepare__ is classmethod — base returns plain dict
print(type.__prepare__("X", ()))     # {}
```

---

## Exercise 4 — `metaclass=` & Metaclass Inheritance

```python
PLUGIN_REGISTRY = {}

class PluginMeta(type):
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        if bases:   # skip the base class itself
            PLUGIN_REGISTRY[name] = cls
        return cls

class BasePlugin(metaclass=PluginMeta):
    pass

class AudioPlugin(BasePlugin): pass   # metaclass= not required — inherited
class VideoPlugin(BasePlugin): pass

print(type(AudioPlugin) is PluginMeta)   # True
print(PLUGIN_REGISTRY)


# Metaclass conflict and resolution
class MetaA(type):
    def __new__(mcs, name, bases, ns):
        print(f"  MetaA.__new__({name})")
        return super().__new__(mcs, name, bases, ns)

class MetaB(type):
    def __new__(mcs, name, bases, ns):
        print(f"  MetaB.__new__({name})")
        return super().__new__(mcs, name, bases, ns)

class A(metaclass=MetaA): pass
class B(metaclass=MetaB): pass

try:
    class C(A, B): pass   # TypeError: metaclass conflict
except TypeError as e:
    print(f"Conflict: {e}")

# Fix: create a metaclass that is a subclass of both
class MetaC(MetaA, MetaB): pass

class C(A, B, metaclass=MetaC): pass   # both MetaA and MetaB __new__ run via super()
print(type(C))   # <class 'MetaC'>

# Python's metaclass resolution algorithm:
# 1. Collect metaclass of every base.
# 2. The winner is the most-derived metaclass — the one that is a subclass of all others.
# 3. If no single winner exists (unrelated metaclasses), raise TypeError.


# __init_subclass__ alternative
ISC_REGISTRY = {}

class BasePlugin2:
    def __init_subclass__(cls, category=None, **kwargs):
        super().__init_subclass__(**kwargs)   # cooperative — must be called
        ISC_REGISTRY[cls.__name__] = {"cls": cls, "category": category}

class ImagePlugin(BasePlugin2, category="media"): pass
class TextPlugin(BasePlugin2, category="data"):  pass

print(ISC_REGISTRY)
# Prefer __init_subclass__ when you only need to react to subclass creation.
# Prefer metaclass when you also need __prepare__, __call__, or __instancecheck__.
```

---

## Exercise 5 — MRO Customization

```python
class MROTracerMeta(type):
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        print(f"  MRO for {name}: {[c.__name__ for c in cls.__mro__]}")
        return cls

class Base(metaclass=MROTracerMeta): pass
class Child(Base): pass
class GrandChild(Child): pass


# Inject LogMixin into every class's MRO without it appearing in __bases__
class LogMixin:
    def log(self): return f"[LOG] {type(self).__name__}"

class MROInjectMeta(type):
    def mro(cls):
        original = super().mro()
        if LogMixin not in original and cls.__name__ != "LogMixin":
            idx = original.index(object)
            return original[:idx] + [LogMixin] + original[idx:]
        return original

class MyBase(metaclass=MROInjectMeta): pass
class MyChild(MyBase): pass

obj = MyChild()
print(obj.log())                      # [LOG] MyChild — LogMixin was injected
print(LogMixin in MyChild.__mro__)    # True
print(LogMixin in MyChild.__bases__)  # False — not in __bases__, only in __mro__

# mro() on the metaclass is what Python calls to build __mro__.
# Overriding it gives full control, but breaking super() chains is easy —
# always ensure object appears exactly once at the end.


# MRO validator — reject shadowing methods with different signatures
class MROValidatorMeta(type):
    def __new__(mcs, name, bases, namespace):
        for attr_name, val in namespace.items():
            if attr_name.startswith("__") or not callable(val):
                continue
            for base in bases:
                parent_val = getattr(base, attr_name, None)
                if parent_val and callable(parent_val):
                    try:
                        if (inspect.signature(val) != inspect.signature(parent_val)):
                            raise TypeError(
                                f"{name}.{attr_name} shadows {base.__name__}.{attr_name} "
                                f"with a different signature"
                            )
                    except (ValueError, TypeError):
                        pass   # some builtins don't expose signatures
        return super().__new__(mcs, name, bases, namespace)
```

---

## Exercise 6 — Abstract Base Classes via Metaclasses

```python
from abc import ABCMeta, ABC, abstractmethod


def abstract(fn):
    """Marker decorator for abstract methods."""
    fn.__is_abstract__ = True
    return fn


class SimpleABCMeta(type):
    def __new__(mcs, name, bases, namespace):
        # Collect abstract methods: new ones + inherited ones not yet overridden
        abstract_methods = {
            k for k, v in namespace.items()
            if callable(v) and getattr(v, "__is_abstract__", False)
        }
        for base in bases:
            abstract_methods |= getattr(base, "__abstract_methods__", set())
        # Remove any that are now concretely implemented in this class
        for k, v in namespace.items():
            if callable(v) and not getattr(v, "__is_abstract__", False):
                abstract_methods.discard(k)

        cls = super().__new__(mcs, name, bases, namespace)
        cls.__abstract_methods__ = frozenset(abstract_methods)
        return cls

    def __call__(cls, *args, **kwargs):
        if cls.__abstract_methods__:
            raise TypeError(
                f"Can't instantiate abstract class {cls.__name__} "
                f"with abstract methods: {cls.__abstract_methods__}"
            )
        return super().__call__(*args, **kwargs)


class Shape(metaclass=SimpleABCMeta):
    @abstract
    def area(self): pass

    @abstract
    def perimeter(self): pass

try:
    Shape()
except TypeError as e:
    print(e)

class Circle(Shape):
    def __init__(self, r): self.r = r
    def area(self): return 3.14159 * self.r ** 2
    def perimeter(self): return 2 * 3.14159 * self.r

print(Circle(5).area())   # 78.53...


# Real ABCMeta usage
class Serializable(ABC):
    @abstractmethod
    def serialize(self) -> bytes: ...

    @abstractmethod
    def deserialize(self, data: bytes): ...

class JsonSerial(Serializable):
    def serialize(self): return b'{"json": true}'
    def deserialize(self, data): return data

print(JsonSerial().serialize())

# Virtual subclass — isinstance works but abstract methods are NOT enforced
class ThirdParty:
    def serialize(self): return b"third"
    def deserialize(self, data): return data

Serializable.register(ThirdParty)
print(isinstance(ThirdParty(), Serializable))   # True
```

---

## Exercise 7 — Singleton Enforcement

```python
import threading, pickle


class SingletonMeta(type):
    _instances = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]


class Config(metaclass=SingletonMeta):
    def __init__(self, value=0): self.value = value

class AppConfig(Config): pass   # separate singleton per subclass

c1 = Config()
c2 = Config()
print(c1 is c2)                   # True
print(Config() is AppConfig())    # False — independent per class


# Thread-safe singleton
class ThreadSafeSingletonMeta(type):
    _instances = {}
    _locks = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls._locks:
            cls._locks[cls] = threading.Lock()
        with cls._locks[cls]:
            if cls not in cls._instances:
                cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class SafeConfig(metaclass=ThreadSafeSingletonMeta):
    def __init__(self): self.value = 42

results = []
def create(): results.append(SafeConfig())
threads = [threading.Thread(target=create) for _ in range(10)]
for t in threads: t.start()
for t in threads: t.join()
print(all(r is results[0] for r in results))   # True


# Resetable singleton (useful in tests)
class ResetableSingletonMeta(type):
    _instances = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

    def _reset_instance(cls):
        cls._instances.pop(cls, None)

class TestConfig(metaclass=ResetableSingletonMeta):
    def __init__(self): self.value = id(self)

t1 = TestConfig()
TestConfig._reset_instance()
t2 = TestConfig()
print(t1 is t2)   # False — reset worked


# Fix pickle: naive singletons break because unpickling creates a new instance
class PickleSafeSingleton(metaclass=SingletonMeta):
    def __reduce__(self):
        return (self.__class__, ())   # deserialises by calling cls() → hits cache

ps = PickleSafeSingleton()
restored = pickle.loads(pickle.dumps(ps))
print(restored is ps)   # True — singleton preserved across pickle round-trip

# Borg pattern (alternative): shared __dict__ rather than shared identity
class Borg:
    _shared = {}
    def __init__(self): self.__dict__ = Borg._shared

b1, b2 = Borg(), Borg()
b1.x = 99
print(b2.x)       # 99 — shared state
print(b1 is b2)   # False — different objects, same dict
```

---

## Exercise 8 — Attribute Validation & Registration

```python
class ValidationError(Exception): pass


class Field:
    def __init__(self, index=False, column=None):
        self.index = index
        self.column = column
        self.name = None

    def __set_name__(self, owner, name):
        self.name = name
        if self.column is None:
            self.column = name

    def __get__(self, obj, objtype=None):
        if obj is None: return self
        return obj.__dict__.get(self.name)

    def __set__(self, obj, value):
        obj.__dict__[self.name] = value

    def validate(self, value): pass


class CharField(Field):
    def __init__(self, max_length=255, **kw):
        super().__init__(**kw)
        self.max_length = max_length

    def validate(self, value):
        if not isinstance(value, str):
            raise ValidationError(f"{self.name}: expected str")
        if len(value) > self.max_length:
            raise ValidationError(f"{self.name}: max_length={self.max_length}, got {len(value)}")


class IntField(Field):
    def __init__(self, min_val=None, max_val=None, **kw):
        super().__init__(**kw)
        self.min_val = min_val
        self.max_val = max_val

    def validate(self, value):
        if not isinstance(value, int):
            raise ValidationError(f"{self.name}: expected int")
        if self.min_val is not None and value < self.min_val:
            raise ValidationError(f"{self.name}: min={self.min_val}, got {value}")
        if self.max_val is not None and value > self.max_val:
            raise ValidationError(f"{self.name}: max={self.max_val}, got {value}")


class FloatField(Field):
    def validate(self, value):
        if not isinstance(value, (int, float)):
            raise ValidationError(f"{self.name}: expected float")


class ModelMeta(type):
    def __new__(mcs, name, bases, namespace):
        fields, columns_seen, indexes = {}, {}, []

        for attr_name, val in list(namespace.items()):
            if not isinstance(val, Field): continue
            if attr_name.startswith("_"):
                raise TypeError(f"Field name cannot start with '_': {attr_name!r}")
            col = val.column or attr_name
            if col in columns_seen:
                raise TypeError(f"Duplicate column '{col}' in {name}")
            columns_seen[col] = attr_name
            fields[attr_name] = val
            if val.index:
                indexes.append(attr_name)

        cls = super().__new__(mcs, name, bases, namespace)
        cls._fields = fields
        cls._indexes = indexes

        fnames = list(fields.keys())

        def __init__(self, **kwargs):
            for fn in fnames:
                object.__setattr__(self, fn, kwargs.get(fn))
        def __repr__(self):
            parts = ", ".join(f"{f}={getattr(self, f)!r}" for f in fnames)
            return f"{type(self).__name__}({parts})"
        def __eq__(self, other):
            if type(self) is not type(other): return NotImplemented
            return all(getattr(self, f) == getattr(other, f) for f in fnames)

        cls.__init__ = __init__
        cls.__repr__ = __repr__
        cls.__eq__ = __eq__
        return cls

    def __call__(cls, **kwargs):
        # Validate ALL fields before creating the instance — collect all errors
        errors = []
        for fname, field in cls._fields.items():
            try:
                field.validate(kwargs.get(fname))
            except ValidationError as e:
                errors.append(str(e))
        if errors:
            raise ValidationError("Validation failed:\n" + "\n".join(f"  • {e}" for e in errors))
        instance = cls.__new__(cls)
        instance.__init__(**kwargs)
        return instance


class Model(metaclass=ModelMeta): pass

class User(Model):
    name  = CharField(max_length=100, index=True)
    age   = IntField(min_val=0, max_val=150)
    score = FloatField()

u = User(name="Alice", age=30, score=9.5)
print(repr(u))                     # User(name='Alice', age=30, score=9.5)
print(User._indexes)               # ['name']

try:
    User(name="x" * 200, age=-1, score=1.0)
except ValidationError as e:
    print(e)
# Validation failed:
#   • name: max_length=100, got 200
#   • age: min=0, got -1
```

---

## Exercise 9 — Dynamic Method Addition

```python
import functools, time


class TestSuiteMeta(type):
    def __new__(mcs, name, bases, namespace):
        tests = {k: v for k, v in namespace.items() if k.startswith("test_")}

        for test_name, test_fn in tests.items():
            def make_runner(fn, tname):
                @functools.wraps(fn)
                def runner(self):
                    t0 = time.perf_counter()
                    try:
                        fn(self)
                        ms = (time.perf_counter() - t0) * 1000
                        print(f"  PASS  {tname} ({ms:.1f}ms)")
                        return True
                    except Exception as exc:
                        print(f"  FAIL  {tname}: {exc}")
                        return False
                return runner
            namespace[test_name] = make_runner(test_fn, test_name)
            # factory function make_runner is CRITICAL here — without it,
            # all closures would capture the last value of test_name (closure trap)

        def run_all(self):
            results = [getattr(self, t)() for t in namespace if t.startswith("test_")]
            print(f"Results: {sum(results)}/{len(results)} passed")
        namespace["run_all"] = run_all

        return super().__new__(mcs, name, bases, namespace)


class MyTests(metaclass=TestSuiteMeta):
    def test_addition(self): assert 1 + 1 == 2
    def test_failure(self):  assert 1 == 2, "expected"
    def test_strings(self):  assert "hi".upper() == "HI"

MyTests().run_all()


class APIMeta(type):
    def __new__(mcs, name, bases, namespace):
        for method_name, (verb, path) in namespace.get("ENDPOINTS", {}).items():
            def make_method(v, p):
                def api_method(self, **kwargs):
                    return {"verb": v, "path": p, "params": kwargs}
                api_method.__name__ = method_name
                return api_method
            namespace[method_name] = make_method(verb, path)
        return super().__new__(mcs, name, bases, namespace)


class GithubClient(metaclass=APIMeta):
    ENDPOINTS = {
        "get_user":    ("GET",  "/users/{user}"),
        "create_repo": ("POST", "/repos"),
    }

print(GithubClient().get_user(user="alice"))


class LoggingMeta(type):
    def __new__(mcs, name, bases, namespace):
        for attr_name, val in list(namespace.items()):
            if callable(val) and not attr_name.startswith("__"):
                def make_logged(fn, aname, cname):
                    @functools.wraps(fn)
                    def logged(self, *args, **kwargs):
                        print(f"[CALL]   {cname}.{aname}{args}")
                        result = fn(self, *args, **kwargs)
                        print(f"[RETURN] {result!r}")
                        return result
                    return logged
                namespace[attr_name] = make_logged(val, attr_name, name)
        return super().__new__(mcs, name, bases, namespace)


class Calculator(metaclass=LoggingMeta):
    def add(self, a, b): return a + b

Calculator().add(3, 4)
```

---

## Exercise 10 — `__instancecheck__` / `__subclasscheck__`

```python
class ProtocolMeta(type):
    def __instancecheck__(cls, instance):
        """Structural isinstance: True if instance has all required methods."""
        required = getattr(cls, "_required_methods", [])
        return all(callable(getattr(instance, m, None)) for m in required)

    def __subclasscheck__(cls, subclass):
        """Structural issubclass: True if subclass defines all required methods."""
        if not isinstance(subclass, type):
            raise TypeError(f"issubclass() arg 1 must be a class")
        required = getattr(cls, "_required_methods", [])
        return all(callable(getattr(subclass, m, None)) for m in required)


class Swimmable(metaclass=ProtocolMeta):
    _required_methods = ["swim"]

class Flyable(metaclass=ProtocolMeta):
    _required_methods = ["fly"]

class Runnable(metaclass=ProtocolMeta):
    _required_methods = ["run"]


class Duck:
    def swim(self): return "swimming"
    def fly(self):  return "flying"

duck = Duck()
print(isinstance(duck, Swimmable))   # True
print(isinstance(duck, Flyable))     # True
print(isinstance(duck, Runnable))    # False — no run()


# CompositeProtocol: And(Swimmable, Flyable)
class AndMeta(type):
    def __instancecheck__(cls, instance):
        return all(isinstance(instance, p) for p in cls._protocols)

def make_and(*protocols):
    return AndMeta("And", (), {"_protocols": protocols})

SwimmingFlier = make_and(Swimmable, Flyable)
print(isinstance(duck, SwimmingFlier))        # True
print(isinstance(Duck(), SwimmingFlier))      # True


# Why __instancecheck__ on a class body doesn't work:
class FakeProtocol:
    def __instancecheck__(self, instance):   # defined on the class — NOT the metaclass
        return True

# isinstance(x, FakeProtocol) checks type(FakeProtocol).__instancecheck__
# type(FakeProtocol) is `type` — its __instancecheck__ does normal subclass checking
print(isinstance("hello", FakeProtocol))   # False — our override is ignored
# Lesson: __instancecheck__ and __subclasscheck__ MUST live on the metaclass.
```

---

## Exercise 11 — `__call__` on Metaclasses

```python
import weakref


class PoolMeta(type):
    def __init__(cls, name, bases, namespace):
        super().__init__(name, bases, namespace)
        cls._pool = []

    def __call__(cls, *args, **kwargs):
        if cls._pool:
            print(f"  Reusing pooled {cls.__name__}")
            return cls._pool.pop()
        return super().__call__(*args, **kwargs)


class PooledConnection(metaclass=PoolMeta):
    def __init__(self, host="localhost"):
        self.host = host

    def release(self):
        type(self)._pool.append(self)

c1 = PooledConnection("db1")
c1.release()           # returns to pool
c2 = PooledConnection() # reuses c1
print(c1 is c2)        # True


class FlyweightMeta(type):
    def __init__(cls, name, bases, namespace):
        super().__init__(name, bases, namespace)
        cls._cache = weakref.WeakValueDictionary()

    def __call__(cls, *args, **kwargs):
        key = (args, tuple(sorted(kwargs.items())))
        if key in cls._cache:
            return cls._cache[key]
        instance = super().__call__(*args, **kwargs)
        cls._cache[key] = instance
        return instance


class Point(metaclass=FlyweightMeta):
    def __init__(self, x, y): self.x, self.y = x, y

p1 = Point(1, 2)
p2 = Point(1, 2)
print(p1 is p2)   # True — interned by arguments


class CoercingMeta(type):
    """ClassName(x) returns x unchanged if x is already an instance of ClassName."""
    def __call__(cls, value, *args, **kwargs):
        if isinstance(value, cls):
            return value
        return super().__call__(value, *args, **kwargs)


class Degrees(metaclass=CoercingMeta):
    def __init__(self, value): self.value = float(value) % 360

d1 = Degrees(90)
d2 = Degrees(d1)   # already a Degrees
print(d1 is d2)    # True — no double-wrapping

# Full __call__ chain (with print statements to prove order):
# MetaMeta.__call__ → MyClass.__new__ → MyClass.__init__
# Overriding __call__ on the metaclass affects class instantiation.
# Overriding __call__ on the class only affects calling instances of that class.
```

---

## Exercise 12 — `__init_subclass__` vs Metaclasses

```python
# Side-by-side: same plugin registry both ways

# --- Metaclass approach ---
META_REGISTRY = {}

class PluginMeta(type):
    def __new__(mcs, name, bases, ns):
        cls = super().__new__(mcs, name, bases, ns)
        if bases:
            META_REGISTRY[name] = cls
        return cls

class BasePluginMeta(metaclass=PluginMeta): pass
class PlugA(BasePluginMeta): pass
class PlugB(BasePluginMeta): pass
print("Meta:", list(META_REGISTRY))

# --- __init_subclass__ approach — ~70% less code ---
ISC_REGISTRY = {}

class BasePlugin:
    def __init_subclass__(cls, version=None, **kwargs):
        super().__init_subclass__(**kwargs)   # MUST call super() for cooperative chains
        ISC_REGISTRY[cls.__name__] = {"cls": cls, "version": version}

class PlugC(BasePlugin, version="1.0"): pass   # version kwarg forwarded
class PlugD(BasePlugin, version="2.0"): pass
print("ISC:", list(ISC_REGISTRY))


# Enforce schema class attribute at subclass definition time
class Validated:
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        if not hasattr(cls, "schema") or not isinstance(cls.schema, dict):
            raise TypeError(f"{cls.__name__} must define 'schema: dict'")

class GoodModel(Validated):
    schema = {"name": str}

try:
    class BadModel(Validated): pass
except TypeError as e:
    print(e)


# Decision matrix (in comments):
# __init_subclass__ CAN replace metaclass for:
#   ✓ Subclass auto-registration
#   ✓ Enforcing class-level attributes on subclasses
#   ✓ Receiving keyword arguments from class definition
# __init_subclass__ CANNOT replace metaclass for:
#   ✗ __prepare__ (namespace customisation before class body)
#   ✗ __call__ (controlling instance creation)
#   ✗ __instancecheck__ / __subclasscheck__
#   ✗ MRO overriding
#   ✗ Injecting into class namespace before class creation
#   ✗ Metaclass conflict resolution
```

---

## Exercise 13 — Cooperative Metaclasses

```python
class MetaX(type):
    def __new__(mcs, name, bases, ns):
        ns["_from_x"] = True
        print(f"  MetaX.__new__({name})")
        return super().__new__(mcs, name, bases, ns)   # MUST use super(), not type.__new__

class MetaY(type):
    def __new__(mcs, name, bases, ns):
        ns["_from_y"] = True
        print(f"  MetaY.__new__({name})")
        return super().__new__(mcs, name, bases, ns)

class MetaZ(type):
    def __new__(mcs, name, bases, ns):
        ns["_from_z"] = True
        print(f"  MetaZ.__new__({name})")
        return super().__new__(mcs, name, bases, ns)


# Cooperative composed metaclass
class CoopMeta(MetaX, MetaY, MetaZ): pass

print("CoopMeta MRO:", [c.__name__ for c in CoopMeta.__mro__])
# CoopMeta -> MetaX -> MetaY -> MetaZ -> type -> object
# super() in MetaX.__new__ calls MetaY.__new__, then MetaZ.__new__, then type.__new__

class MyClass(metaclass=CoopMeta): pass
print(MyClass._from_x, MyClass._from_y, MyClass._from_z)   # True True True

# Fragile: using type.__new__ directly breaks cooperation
class BrokenMeta(type):
    def __new__(mcs, name, bases, ns):
        return type.__new__(mcs, name, bases, ns)   # bypasses super() chain — BAD

# Fix: always use super().__new__(...)


# mixin_meta factory — resolve conflicts dynamically
def mixin_meta(*metaclasses):
    if len(metaclasses) == 1:
        return metaclasses[0]
    merged_name = "MixedMeta_" + "_".join(m.__name__ for m in metaclasses)
    return type(merged_name, metaclasses, {})

DynamicMeta = mixin_meta(MetaX, MetaY)

class DynClass(metaclass=DynamicMeta): pass
print(DynClass._from_x, DynClass._from_y)   # True True
```

---

## Exercise 14 — Class Decorators vs Metaclasses

```python
import functools, inspect


# ── 1. Auto-repr ──────────────────────────────────────────

# Class decorator version
def auto_repr_decorator(cls):
    params = [p for p in inspect.signature(cls.__init__).parameters if p != "self"]
    def __repr__(self):
        attrs = ", ".join(f"{p}={getattr(self, p, '?')!r}" for p in params)
        return f"{type(self).__name__}({attrs})"
    cls.__repr__ = __repr__
    return cls

@auto_repr_decorator
class Vector:
    def __init__(self, x, y): self.x, self.y = x, y

print(repr(Vector(3, 4)))   # Vector(x=3, y=4)

# Metaclass version — auto-applies to ALL subclasses
class AutoReprMeta(type):
    def __new__(mcs, name, bases, ns):
        cls = super().__new__(mcs, name, bases, ns)
        params = [p for p in inspect.signature(cls.__init__).parameters if p != "self"]
        def __repr__(self):
            attrs = ", ".join(f"{p}={getattr(self, p, '?')!r}" for p in params)
            return f"{type(self).__name__}({attrs})"
        cls.__repr__ = __repr__
        return cls

class Vector2(metaclass=AutoReprMeta):
    def __init__(self, x, y): self.x, self.y = x, y

print(repr(Vector2(3, 4)))
# Decorator: simpler, opt-in per class.
# Metaclass: automatic for all subclasses — no re-applying needed.
# Decorator FAILS when: you want subclasses to automatically inherit the behaviour.


# ── 5. @class_decorator_to_meta ──────────────────────────

def class_decorator_to_meta(decorator_fn):
    """Convert any class decorator into an equivalent metaclass.
    The metaclass applies the decorator to every class (and subclass) created.
    """
    class _Meta(type):
        def __new__(mcs, name, bases, ns):
            cls = super().__new__(mcs, name, bases, ns)
            return decorator_fn(cls)
    _Meta.__name__ = f"MetaFor_{decorator_fn.__name__}"
    return _Meta

AutoReprFromDecorator = class_decorator_to_meta(auto_repr_decorator)

class Point3D(metaclass=AutoReprFromDecorator):
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z

class ColorPoint3D(Point3D):   # inherits metaclass — auto-repr applied automatically
    def __init__(self, x, y, z, color):
        super().__init__(x, y, z)
        self.color = color

print(repr(Point3D(1, 2, 3)))           # Point3D(x=1, y=2, z=3)
print(repr(ColorPoint3D(1, 2, 3, "red")))  # ColorPoint3D(x=1, y=2, z=3, color='red')
```

---

## Bonus — `ModelForge`

```python
from abc import ABCMeta
import threading


class FieldOrderedNamespace(dict):
    """Custom namespace that records field definition order and rejects duplicates."""
    def __init__(self):
        super().__init__()
        self._field_order = []

    def __setitem__(self, key, value):
        if isinstance(value, Field) and key in self:
            raise TypeError(f"Duplicate field: {key!r}")
        if isinstance(value, Field):
            self._field_order.append(key)
        super().__setitem__(key, value)


class ModelMeta(ABCMeta):
    """Cooperative metaclass: ABCMeta + validation + registration."""

    @classmethod
    def __prepare__(mcs, name, bases, abstract=False, **kwargs):
        return FieldOrderedNamespace()

    def __new__(mcs, name, bases, namespace, abstract=False, **kwargs):
        fields, columns_seen, indexes = {}, {}, []

        for fname, val in list(namespace.items()):
            if not isinstance(val, Field): continue
            if fname.startswith("_"):
                raise TypeError(f"Field '{fname}' cannot start with '_'")
            col = val.column or fname
            if col in columns_seen:
                raise TypeError(f"Duplicate column '{col}'")
            columns_seen[col] = fname
            fields[fname] = val
            if val.index:
                indexes.append(fname)

        field_order = getattr(namespace, "_field_order", list(fields.keys()))
        cls = super().__new__(mcs, name, bases, dict(namespace), **kwargs)
        cls._fields = fields
        cls._field_order = field_order
        cls._indexes = indexes
        cls._hooks = {"pre_save": [], "post_save": []}
        cls._abstract_model = abstract

        if not abstract and bases:
            fnames = field_order

            def __init__(self, **kw):
                for fn in fnames:
                    object.__setattr__(self, fn, kw.get(fn))

            def __repr__(self):
                parts = ", ".join(f"{f}={getattr(self, f)!r}" for f in fnames)
                return f"{type(self).__name__}({parts})"

            def __eq__(self, other):
                if type(self) is not type(other): return NotImplemented
                return all(getattr(self, f) == getattr(other, f) for f in fnames)

            cls.__init__ = __init__
            cls.__repr__ = __repr__
            cls.__eq__   = __eq__

        return cls

    def __call__(cls, **kwargs):
        """Validate all fields before creating instance — collect all errors."""
        errors = []
        for fname, field in cls._fields.items():
            try:
                field.validate(kwargs.get(fname))
            except ValidationError as e:
                errors.append(str(e))
        if errors:
            raise ValidationError("Validation failed:\n" + "\n".join(f"  • {e}" for e in errors))
        instance = cls.__new__(cls)
        instance.__init__(**kwargs)
        return instance

    def __instancecheck__(cls, instance):
        """Structural check: True if instance has all required fields."""
        if type.__instancecheck__(cls, instance):
            return True
        return all(hasattr(instance, f) for f in cls._fields)


# Singleton for the registry itself
class SingletonMeta(type):
    _instances = {}
    def __call__(cls, *a, **kw):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*a, **kw)
        return cls._instances[cls]

class ModelForge(metaclass=SingletonMeta):
    registry = {}


class Model(metaclass=ModelMeta, abstract=True):
    def __init_subclass__(cls, abstract=False, **kwargs):
        super().__init_subclass__(**kwargs)
        if not abstract:
            ModelForge.registry[cls.__name__] = cls


# --- Concrete model ---
class User(Model, abstract=False):
    table_name = "users"
    name  = CharField(max_length=100, index=True)
    age   = IntField(min_val=0, max_val=150)
    email = CharField(max_length=200, column="email_address")

u = User(name="Alice", age=30, email="alice@example.com")
print(repr(u))
# User(name='Alice', age=30, email='alice@example.com')

print(ModelForge.registry)
# {'User': <class '__main__.User'>}

print(isinstance(u, Model))
# True — structural check via __instancecheck__

try:
    User(name="x" * 200, age=-1, email="")
except ValidationError as e:
    print(e)
# Validation failed:
#   • name: max_length=100, got 200
#   • age: min=0, got -1
```
