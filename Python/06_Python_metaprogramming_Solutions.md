# Python Metaprogramming — Solutions

---

## Exercise 1 — `inspect` Module Deep Dive

```python
import inspect


def create_handler(
    user_id: int,
    action: str,
    payload: dict = None,
    *tags: str,
    dry_run: bool = False,
    **options
) -> dict:
    """Handle a user action.

    Args:
        user_id: The acting user's ID.
        action: The action name.
        payload: Optional data payload.

    Returns:
        Result dict with status and data.
    """
    return {"user_id": user_id, "action": action}


# 1. Signature extraction
sig = inspect.signature(create_handler)
for name, param in sig.parameters.items():
    ann = param.annotation if param.annotation is not param.empty else "none"
    default = param.default if param.default is not param.empty else "required"
    print(f"  {name}: kind={param.kind.name}, annotation={ann}, default={default}")


# 2. Categorise class members
class SampleClass:
    class_attr = "hello"

    def __init__(self): self.x = 1
    def instance_method(self): return self.x

    @classmethod
    def class_method(cls): return cls

    @staticmethod
    def static_method(): return 42

    @property
    def prop(self): return self.x * 2


members = {"methods": [], "properties": [], "classmethods": [],
           "staticmethods": [], "dunders": [], "attributes": []}

for name, val in inspect.getmembers(SampleClass):
    if name.startswith("__") and name.endswith("__"):
        members["dunders"].append(name)
    elif isinstance(inspect.getattr_static(SampleClass, name), property):
        members["properties"].append(name)
    elif isinstance(inspect.getattr_static(SampleClass, name), classmethod):
        members["classmethods"].append(name)
    elif isinstance(inspect.getattr_static(SampleClass, name), staticmethod):
        members["staticmethods"].append(name)
    elif callable(val):
        members["methods"].append(name)
    else:
        members["attributes"].append(name)

for k, v in members.items():
    if v: print(f"  {k}: {v}")


# 3. getsource — raises for built-in C functions
src = inspect.getsource(create_handler)
print(f"  source lines: {len(src.splitlines())}")
try:
    inspect.getsource(len)
except (OSError, TypeError) as e:
    print(f"  built-in: {e}")


# 4. getdoc vs __doc__: getdoc strips leading indentation
def indented():
    """
    First line.
    Second line.
    """
    pass

print(repr(indented.__doc__))          # '\n    First line.\n    Second line.\n    '
print(repr(inspect.getdoc(indented)))  # 'First line.\nSecond line.'


# 5. Stack inspection inside nested calls
def a():
    def b():
        def c():
            for frame_info in inspect.stack():
                print(f"  {frame_info.function} @ "
                      f"{frame_info.filename.split('/')[-1]}:{frame_info.lineno}")
        c()
    b()
a()


# 6. function_schema
def function_schema(fn) -> dict:
    sig = inspect.signature(fn)
    try:
        src_file = inspect.getfile(fn)
        src_line = inspect.getsourcelines(fn)[1]
    except (OSError, TypeError):
        src_file = src_line = None
    return {
        "name": fn.__name__,
        "doc":  inspect.getdoc(fn),
        "params": [
            {
                "name":       name,
                "annotation": str(p.annotation) if p.annotation is not p.empty else None,
                "default":    None if p.default is p.empty else p.default,
                "kind":       p.kind.name,
            }
            for name, p in sig.parameters.items()
        ],
        "return_annotation": (
            str(sig.return_annotation)
            if sig.return_annotation is not sig.empty else None
        ),
        "source_file": src_file,
        "source_line": src_line,
    }

schema = function_schema(create_handler)
print(f"  schema: name={schema['name']}, params={len(schema['params'])}")
```

---

## Exercise 2 — `dir()`, `vars()`, `callable()`

```python
import inspect


# 1. Three-way difference
class SlottedClass:
    __slots__ = ("x", "y")
    def __init__(self): self.x = 1; self.y = 2

s = SlottedClass()
try:
    vars(s)   # TypeError — no __dict__ on slots-only instances
except TypeError as e:
    print(f"  vars on slots: {e}")

# dir() — walks full MRO, includes inherited attrs, class attrs, dunders
# vars(obj) — returns obj.__dict__ (fails on slots, built-ins)
# obj.__dict__ — raw instance dict only, KeyError if absent


# 2. explore()
class Inner:
    cfg = {"a": 1}
    def __init__(self): self.value = 42
    def method(self): pass
    @property
    def computed(self): return self.value * 2

def explore(obj) -> dict:
    result = {"callables": [], "properties": [], "data": [], "dunders": []}
    for name in dir(obj):
        if name.startswith("__") and name.endswith("__"):
            result["dunders"].append(name)
            continue
        # Check for property anywhere in MRO without triggering __get__
        for klass in type(obj).__mro__:
            if name in vars(klass) and isinstance(vars(klass)[name], property):
                result["properties"].append(name)
                break
        else:
            val = getattr(obj, name, None)
            if callable(val):
                result["callables"].append(name)
            else:
                result["data"].append(name)
    return result

explored = explore(Inner())
print(f"  callables: {explored['callables']}")
print(f"  properties: {explored['properties']}")
print(f"  data: {explored['data']}")


# 3. dir() walks MRO; vars() does not
class GrandParent:
    gp_attr = "gp"

class Parent(GrandParent):
    p_attr = "p"

class Child(Parent):
    c_attr = "c"

c = Child()
print(f"  'gp_attr' in dir(c): {'gp_attr' in dir(c)}")         # True — MRO walk
print(f"  'gp_attr' in vars(Child): {'gp_attr' in vars(Child)}")  # False — direct only


# 4. callable() with all kinds
class CallableClass:
    def __call__(self): pass

for obj, label in [
    (lambda: None,       "lambda"),
    (len,                "builtin"),
    (CallableClass,      "class"),
    (CallableClass(),    "instance with __call__"),
    (str.upper,          "unbound method"),
    ("hello".upper,      "bound method"),
]:
    print(f"  callable({label}): {callable(obj)}")


# 5. attribute_origin
def attribute_origin(obj, name) -> type:
    for klass in type(obj).__mro__:
        if name in vars(klass):
            return klass
    raise AttributeError(f"{name!r} not found in MRO")

class Base:
    x = 10

class Child(Base):
    y = 20

ch = Child()
print(f"  x origin: {attribute_origin(ch, 'x').__name__}")   # Base
print(f"  y origin: {attribute_origin(ch, 'y').__name__}")   # Child


# 6. __dir__ customisation
class SmartDir:
    _hidden = {"_private", "_internal"}
    _virtual_attrs = ["virtual_a", "virtual_b"]
    _private = "secret"

    def __dir__(self):
        normal = [n for n in object.__dir__(self) if n not in self._hidden]
        return sorted(normal + self._virtual_attrs)

sd = SmartDir()
print(f"  '_private' in dir: {'_private' in dir(sd)}")    # False — hidden
print(f"  'virtual_a' in dir: {'virtual_a' in dir(sd)}")  # True  — injected
```

---

## Exercise 3 — `getattr` / `setattr` / `delattr` / `hasattr`

```python
import fnmatch
import inspect

MISSING = object()


# 1. DynamicProxy — __getattr__ fires ONLY when normal attribute lookup fails
# __getattribute__ fires on EVERY attribute access, including existing ones
class DynamicProxy:
    def __init__(self, target):
        object.__setattr__(self, "_target", target)   # bypass our own __setattr__

    def __getattr__(self, name):
        target = object.__getattribute__(self, "_target")
        try:
            return getattr(target, name)
        except AttributeError:
            raise AttributeError(
                f"{type(self).__name__}: target {type(target).__name__!r} "
                f"has no attribute {name!r}"
            )

class Backend:
    def query(self): return "db result"

proxy = DynamicProxy(Backend())
print(proxy.query())   # db result
try:
    proxy.missing
except AttributeError as e:
    print(e)


# 2. RoutingProxy
class RoutingProxy:
    def __init__(self, routes: dict):
        self._routes = routes   # {pattern: backend}

    def __getattr__(self, name):
        for pattern, backend in self._routes.items():
            if fnmatch.fnmatch(name, pattern):
                # Strip prefix (e.g. "db_users" → "users") for the backend call
                suffix = name.split("_", 1)[1] if "_" in name else name
                return getattr(backend, suffix)
        raise AttributeError(f"No route matched for {name!r}")

class DBBackend:
    def users(self): return ["alice", "bob"]

class CacheBackend:
    def users(self): return ["cached_alice"]

router = RoutingProxy({"db_*": DBBackend(), "cache_*": CacheBackend()})
print(router.db_users())     # ['alice', 'bob']
print(router.cache_users())  # ['cached_alice']


# 3. apply_config
def apply_config(obj, config: dict):
    for key, value in config.items():
        if key.startswith("_"):
            continue
        try:
            setattr(obj, key, value)
        except AttributeError:
            pass   # read-only or slot mismatch — skip silently

class Config:
    def __init__(self): self.host = "localhost"; self.port = 5432

cfg = Config()
apply_config(cfg, {"host": "prod.db", "port": 5433, "_secret": "skip"})
print(f"  host: {cfg.host}, port: {cfg.port}")


# 4. safe_call
# hasattr is equivalent to: try: getattr(obj, name); return True except AttributeError: return False
def safe_call(obj, method_name, *args, **kwargs):
    if hasattr(obj, method_name):
        return getattr(obj, method_name)(*args, **kwargs)
    return None

print(safe_call(cfg, "nonexistent"))   # None — no crash


# 5. deep_getattr
def deep_getattr(obj, dotted_path: str, default=MISSING):
    parts = dotted_path.split(".")
    for part in parts:
        try:
            obj = getattr(obj, part)
        except AttributeError:
            if default is MISSING:
                raise
            return default
    return obj

class DB:
    class host:
        port = 5432

print(deep_getattr(DB, "host.port"))          # 5432
print(deep_getattr(DB, "host.missing", -1))   # -1


# 6. hasattr footgun: property that raises AttributeError internally
class BuggyModel:
    @property
    def status(self):
        # Real bug: tries to access a non-existent attribute, raises AttributeError
        raise AttributeError("internal bug — status lookup failed!")

bm = BuggyModel()
print(f"  hasattr(bm, 'status'): {hasattr(bm, 'status')}")   # False — bug swallowed!

# Safe diagnostic: inspect.getattr_static doesn't invoke __get__
static_val = inspect.getattr_static(bm, "status", None)
print(f"  is a property (getattr_static): {isinstance(static_val, property)}")  # True
# Now we know it EXISTS but its getter raises — a real bug, not a missing attribute
```

---

## Exercise 4 — `eval`, `exec`, and `compile`

```python
import ast, timeit


# 1. safe_eval with restricted namespace
def safe_eval(expr: str, context: dict):
    restricted = {k: v for k, v in context.items() if not k.startswith("_")}
    restricted["__builtins__"] = {}   # strip all built-ins
    return eval(expr, restricted)

print(safe_eval("x * 2 + y", {"x": 3, "y": 4}))   # 10

try:
    safe_eval("__import__('os').system('ls')", {})
except (NameError, KeyError) as e:
    print(f"  import blocked: {type(e).__name__}: {e}")


# 2. eval vs exec: eval = expressions only
try:
    eval("x = 1")   # assignment is a statement, not an expression
except SyntaxError as e:
    print(f"  eval statement: SyntaxError — {e.msg}")
# eval("1 + 2")  ← OK: expression
# eval("x = 1") ← SyntaxError: statement


# 3. exec to dynamically define a function
def make_adder_via_exec(n: int):
    namespace = {}
    exec(f"def adder(x):\n    return x + {n}", namespace)
    return namespace["adder"]

add5 = make_adder_via_exec(5)
print(f"  exec adder: {add5(10)}")   # 15


# 4. make_init with explicit signature (like dataclasses does internally)
import inspect as _inspect

def make_init(fields: list) -> callable:
    params      = ", ".join(fields)
    assignments = "\n    ".join(f"self.{f} = {f}" for f in fields)
    src = f"def __init__(self, {params}):\n    {assignments}"
    namespace = {}
    exec(src, namespace)
    return namespace["__init__"]

init = make_init(["name", "age", "email"])
print(f"  make_init signature: {_inspect.signature(init)}")
# (self, name, age, email) — explicit params, not (**kwargs)


# 5. exec locals pitfall
x = 10
exec("x = 99")
print(f"  exec locals pitfall: x = {x}")
# In CPython, local variable access uses fast slots (LOAD_FAST opcode),
# not dict lookups. exec() writes to a temporary locals dict that is NOT
# the same as the function's fast locals — so the outer x is unchanged.
# (At module level, exec CAN modify the namespace, so results may vary.)


# 6. compile modes
eval_code   = compile("1 + 2", "<expr>", "eval")    # expression → returns value
exec_code   = compile("y = 42", "<stmt>", "exec")   # statements → returns None
single_code = compile("1 + 2", "<repl>", "single")  # prints result like REPL
print(f"  eval code type: {type(eval_code).__name__}")


# 7. Compile once, execute many times
import timeit

code_obj = compile("[x**2 for x in range(100)]", "<bench>", "eval")
t_str    = timeit.timeit("eval('[x**2 for x in range(100)]', {})", number=10000)
t_code   = timeit.timeit("eval(code_obj, {})", globals={"code_obj": code_obj}, number=10000)
print(f"  re-parse each time: {t_str:.3f}s | reuse compiled: {t_code:.3f}s")
# Reusing the compiled object is measurably faster — no parsing/compilation overhead


# 8. AST inspection to reject function calls before eval
def ast_safe_eval(expr: str, context: dict):
    tree = ast.parse(expr, mode="eval")
    for node in ast.walk(tree):
        if isinstance(node, ast.Call):
            raise ValueError(f"Function calls are not allowed: {expr!r}")
    code = compile(tree, "<ast_safe>", "eval")
    return eval(code, {"__builtins__": {}}, context)

print(f"  ast_safe_eval: {ast_safe_eval('x + y * 2', {'x': 1, 'y': 3})}")   # 7

try:
    ast_safe_eval("len(x)", {"x": [1, 2, 3]})
except ValueError as e:
    print(f"  call blocked: {e}")
```

---

## Exercise 5 — Data Descriptors

```python
# Descriptor lookup priority (highest to lowest):
# 1. Data descriptor (has __get__ AND __set__/__delete__)
# 2. Instance __dict__
# 3. Non-data descriptor (has __get__ only)
# 4. Class attribute (plain value on the class)


class DataDesc:
    """Data descriptor: overrides instance __dict__."""
    def __set_name__(self, owner, name): self.name = name
    def __get__(self, obj, objtype=None):
        if obj is None: return self
        return obj.__dict__.get(f"_dd_{self.name}", "data_desc_value")
    def __set__(self, obj, value):
        obj.__dict__[f"_dd_{self.name}"] = value
    def __delete__(self, obj):
        obj.__dict__.pop(f"_dd_{self.name}", None)


class NonDataDesc:
    """Non-data descriptor: can be shadowed by instance __dict__."""
    def __set_name__(self, owner, name): self.name = name
    def __get__(self, obj, objtype=None):
        if obj is None: return self
        return "non_data_desc_value"


class PriorityDemo:
    data     = DataDesc()
    non_data = NonDataDesc()

pd = PriorityDemo()
pd.data = "via_descriptor"
pd.__dict__["non_data"] = "instance_shadow"  # shadows the non-data descriptor

print(f"  data desc (wins over __dict__): {pd.data}")      # via_descriptor
print(f"  non_data (__dict__ shadows it): {pd.non_data}")  # instance_shadow


# Typed descriptor
class Typed:
    def __init__(self, type_):
        self.type_ = type_
        self.name = None

    def __set_name__(self, owner, name): self.name = name

    def __get__(self, obj, objtype=None):
        if obj is None: return self
        return obj.__dict__.get(self.name)

    def __set__(self, obj, value):
        if not isinstance(value, self.type_):
            raise TypeError(
                f"{self.name}: expected {self.type_.__name__}, got {type(value).__name__}"
            )
        obj.__dict__[self.name] = value   # store in instance __dict__


class Person:
    name = Typed(str)
    age  = Typed(int)

p = Person()
p.name = "Alice"
p.age  = 30
print(f"  Typed: name={p.name}, age={p.age}")

try:
    p.age = "thirty"
except TypeError as e:
    print(f"  Type error: {e}")

# Without __set__, writing to instance __dict__ bypasses the descriptor:
# p.__dict__["age"] = "bypass"  would work if Typed had no __set__
# With __set__ present, Python calls the descriptor instead — bypass impossible


# Lazy non-data descriptor
class Lazy:
    """Computes value once, caches in instance __dict__ — subsequent access bypasses descriptor."""
    def __init__(self, factory_fn):
        self.factory_fn = factory_fn
        self.name = None

    def __set_name__(self, owner, name): self.name = name

    def __get__(self, obj, objtype=None):
        if obj is None: return self
        # First access: compute and store in instance __dict__
        # Next access: __dict__ entry is found BEFORE this descriptor (no __set__ = non-data)
        value = self.factory_fn(obj)
        obj.__dict__[self.name] = value
        return value


class Circle:
    def __init__(self, radius): self.radius = radius
    area = Lazy(lambda self: 3.14159 * self.radius ** 2)

c = Circle(5)
print(f"  'area' in __dict__ before: {'area' in c.__dict__}")  # False
_ = c.area
print(f"  'area' in __dict__ after:  {'area' in c.__dict__}")  # True — cached


# Observable descriptor
class Observable:
    def __init__(self, initial=None):
        self.initial = initial
        self.name = None
        self._subscribers = []

    def __set_name__(self, owner, name): self.name = name

    def __get__(self, obj, objtype=None):
        if obj is None: return self
        return obj.__dict__.get(self.name, self.initial)

    def __set__(self, obj, value):
        old = obj.__dict__.get(self.name, self.initial)
        obj.__dict__[self.name] = value
        for cb in self._subscribers:
            cb(obj, old, value)

    def subscribe(self, cb):
        self._subscribers.append(cb)


class Temperature:
    celsius = Observable(0.0)

def on_change(obj, old, new):
    print(f"  Temperature changed: {old} → {new}")

Temperature.celsius.subscribe(on_change)
t = Temperature()
t.celsius = 20.0   # Temperature changed: 0.0 → 20.0
t.celsius = 37.0   # Temperature changed: 20.0 → 37.0
```

---

## Exercise 6 — Non-Data Descriptors & `__set_name__`

```python
import functools, re


# 1. Function descriptor — replicates Python's method binding
class BoundMethod:
    def __init__(self, fn, instance):
        self.fn = fn
        self.instance = instance
        functools.update_wrapper(self, fn)

    def __call__(self, *args, **kwargs):
        return self.fn(self.instance, *args, **kwargs)

    def __repr__(self):
        return f"<BoundMethod {self.fn.__name__} of {self.instance!r}>"


class Function:
    """Non-data descriptor: __get__ binds the function to the instance."""
    def __init__(self, fn):
        self.fn = fn
        functools.update_wrapper(self, fn)

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return BoundMethod(self.fn, obj)

    def __call__(self, *args, **kwargs):
        return self.fn(*args, **kwargs)


class MyClass:
    def raw_greet(self, name):
        return f"Hello {name} from {type(self).__name__}"

    greet = Function(raw_greet)

obj = MyClass()
print(obj.greet("Alice"))         # Hello Alice from MyClass
print(type(obj.greet).__name__)   # BoundMethod


# 2. ClassMethod descriptor from scratch
class ClassMethod:
    def __init__(self, fn): self.fn = fn

    def __get__(self, obj, objtype=None):
        cls = objtype if objtype is not None else type(obj)
        return functools.partial(self.fn, cls)


class WithClassMethod:
    @ClassMethod
    def who_am_i(cls):
        return f"I am {cls.__name__}"

print(WithClassMethod.who_am_i())       # I am WithClassMethod
print(WithClassMethod().who_am_i())     # I am WithClassMethod


# 3. StaticMethod descriptor from scratch
class StaticMethod:
    def __init__(self, fn): self.fn = fn

    def __get__(self, obj, objtype=None):
        return self.fn   # no binding whatsoever


class WithStaticMethod:
    @StaticMethod
    def add(a, b): return a + b

print(WithStaticMethod.add(3, 4))       # 7
print(WithStaticMethod().add(3, 4))     # 7


# 4. AutoSlug using __set_name__
class AutoSlug:
    def __set_name__(self, owner, name):
        self.name = name
        self.storage_name = f"_slug_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None: return self
        return getattr(obj, self.storage_name, None)

    def __set__(self, obj, value: str):
        slug = re.sub(r"[^\w\s-]", "", value.lower())
        slug = re.sub(r"[\s_-]+", "-", slug).strip("-")
        setattr(obj, self.storage_name, slug)


class Article:
    title    = AutoSlug()
    category = AutoSlug()

a = Article()
a.title    = "Hello World! This is a Test"
a.category = "Python & Programming"
print(a.title)      # hello-world-this-is-a-test
print(a.category)   # python-programming


# 5. VersionedAttribute
class VersionedAttribute:
    def __set_name__(self, owner, name):
        self.name = name
        self.history_key = f"_history_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None: return self
        history = getattr(obj, self.history_key, [])
        return history[-1] if history else None

    def __set__(self, obj, value):
        history = list(getattr(obj, self.history_key, []))
        history.append(value)
        object.__setattr__(obj, self.history_key, history)

    def get_history(self, obj) -> list:
        return list(getattr(obj, self.history_key, []))


class Document:
    content = VersionedAttribute()

doc = Document()
doc.content = "v1"
doc.content = "v2"
doc.content = "v3"
print(doc.content)                               # v3
print(Document.content.get_history(doc))         # ['v1', 'v2', 'v3']


# 6. classmethod and staticmethod are non-data descriptors
print(f"classmethod has __set__: {hasattr(classmethod, '__set__')}")   # False
print(f"staticmethod has __set__: {hasattr(staticmethod, '__set__')}")  # False

# Consequence: an instance __dict__ entry CAN shadow them.
# In practice this almost never matters because you'd have to deliberately do
# instance.__dict__["class_method"] = something to create the shadow.
```

---

## Exercise 7 — Monkey Patching

```python
import types
from contextlib import contextmanager


# 1. patch context manager
@contextmanager
def patch(target, attr_name, new_value):
    original = getattr(target, attr_name, None)
    setattr(target, attr_name, new_value)
    try:
        yield
    finally:
        if original is None:
            try: delattr(target, attr_name)
            except AttributeError: pass
        else:
            setattr(target, attr_name, original)


class Service:
    def fetch(self): return "real data"

svc = Service()
print(svc.fetch())                                   # real data

with patch(Service, "fetch", lambda self: "mocked"):
    print(svc.fetch())                               # mocked

print(svc.fetch())                                   # real data — restored


# 2. Adding a method to an existing class — works on pre-existing instances too
class TextProcessor:
    def __init__(self, text): self.text = text

existing = TextProcessor("hello world foo")   # created BEFORE the patch

TextProcessor.word_count = lambda self: len(self.text.split())
print(existing.word_count())   # 3 — works on pre-existing instance


# 3. Instance patching pitfall + fix
class Greeter:
    def hello(self): return "real hello"

g = Greeter()

# Broken: plain function on instance — no automatic self
g.hello = lambda: "patched without self"
print(g.hello())   # "patched without self" — works here only because lambda ignores self

# Fix: types.MethodType binds the instance explicitly
def patched_hello(self): return f"patched with self: {type(self).__name__}"
g.hello = types.MethodType(patched_hello, g)
print(g.hello())   # patched with self: Greeter


# 4. MonkeyPatchRegistry
class MonkeyPatchRegistry:
    def __init__(self):
        self._patches = []   # [(target, attr, original)]

    @contextmanager
    def patch(self, target, attr_name, new_value):
        original = getattr(target, attr_name, None)
        self._patches.append((target, attr_name, original))
        setattr(target, attr_name, new_value)
        try:
            yield
        finally:
            setattr(target, attr_name, original)
            if self._patches and self._patches[-1][0] is target:
                self._patches.pop()

    def restore_all(self):
        for target, attr, original in reversed(self._patches):
            if original is None:
                try: delattr(target, attr)
                except AttributeError: pass
            else:
                setattr(target, attr, original)
        self._patches.clear()

registry = MonkeyPatchRegistry()

class DB:
    def connect(self): return "connected"

with registry.patch(DB, "connect", lambda self: "mock connected"):
    print(DB().connect())   # mock connected

print(DB().connect())       # connected — restored


# 5. MRO propagation
class Animal:
    def speak(self): return "..."

class Dog(Animal): pass       # inherits

class Cat(Animal):
    def speak(self): return "meow"   # overrides

Animal.speak = lambda self: "PATCHED"

print(Dog().speak())   # PATCHED — inherits the patched method
print(Cat().speak())   # meow    — override protects Cat

Animal.speak = lambda self: "..."   # restore


# 6. Dunder patching: instance vs class
class MyList(list): pass

ml = MyList([1, 2, 3])
ml.__len__ = lambda: 99   # on the INSTANCE — silently ignored by len()
print(f"  len(ml) after instance patch: {len(ml)}")   # 3 — ignored

# Python's special method lookup: len(x) → type(x).__len__(x)
# It goes directly to the TYPE, bypassing instance __dict__ entirely.
MyList.__len__ = lambda self: 99   # on the CLASS — now len() uses it
print(f"  len(ml) after class patch:    {len(ml)}")   # 99

del MyList.__len__   # restore
```

---

## Exercise 8 — `functools.partial`, `partialmethod`, `reduce`

```python
import functools, time


# 1. Pre-configured callables
def clamp(value, lo, hi): return max(lo, min(hi, value))

open_utf8  = functools.partial(open, encoding="utf-8", mode="r")
sorted_ci  = functools.partial(sorted, key=str.lower)
clamp_byte = functools.partial(clamp, lo=0, hi=255)

print(clamp_byte(300))                                      # 255
print(clamp_byte(-10))                                      # 0
print(sorted_ci(["Banana", "apple", "Cherry"]))             # ['apple', 'Banana', 'Cherry']


# 2. Introspect partial attributes
print(f"  .func:     {clamp_byte.func.__name__}")   # clamp
print(f"  .args:     {clamp_byte.args}")             # ()
print(f"  .keywords: {clamp_byte.keywords}")         # {'lo': 0, 'hi': 255}
print(f"  __wrapped__: {hasattr(clamp_byte, '__wrapped__')}")  # False
# partial is not a wrapper-based decorator — it holds the original in .func, not __wrapped__


# 3. pipeline via reduce
def pipeline(value, *fns):
    return functools.reduce(lambda v, f: f(v), fns, value)

result = pipeline(
    "  Hello World  ",
    str.strip,
    str.lower,
    lambda s: s.replace(" ", "_"),
)
print(result)   # hello_world
# Equivalent to: str.replace(str.lower(str.strip("  Hello World  ")), " ", "_")


# 4. partialmethod for REST shortcuts
class HTTPClient:
    def _request(self, method: str, path: str, **kwargs):
        return {"method": method, "path": path, "kwargs": kwargs}

    get    = functools.partialmethod(_request, "GET")
    post   = functools.partialmethod(_request, "POST")
    delete = functools.partialmethod(_request, "DELETE")

client = HTTPClient()
print(client.get("/users"))
print(client.post("/users", body={"name": "Alice"}))


# 5. partial vs lambda
add_five_partial = functools.partial(clamp, lo=5, hi=10)
add_five_lambda  = lambda v: clamp(v, lo=5, hi=10)

# partial: inspectable
print(f"  partial.func: {add_five_partial.func.__name__}")
print(f"  partial.keywords: {add_five_partial.keywords}")

# lambda: opaque — can't inspect what function or args are involved
print(f"  lambda __name__: {add_five_lambda.__name__}")   # <lambda>
# Use partial when: introspection, serialisation, or documentation matters
# Use lambda when: one-liner transformation that won't be inspected


# 6. memoize_with_ttl factory using partial
def _memoize(fn, ttl_seconds: float):
    cache = {}

    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        key = (args, tuple(sorted(kwargs.items())))
        entry = cache.get(key)
        if entry and (time.monotonic() - entry["ts"]) < ttl_seconds:
            return entry["val"]
        val = fn(*args, **kwargs)
        cache[key] = {"val": val, "ts": time.monotonic()}
        return val

    wrapper.cache_clear = cache.clear
    return wrapper

# memoize_with_ttl(n) returns a decorator
memoize_with_ttl = lambda ttl: functools.partial(_memoize, ttl_seconds=ttl)
cache_1s  = memoize_with_ttl(1)
cache_60s = memoize_with_ttl(60)

call_count = [0]
def expensive(x):
    call_count[0] += 1
    return x * 2

cached = cache_60s(expensive)
print(cached(5), call_count[0])   # 10  1
print(cached(5), call_count[0])   # 10  1 — cached, no re-computation
```

---

## Exercise 9 — `dis` Module & Code Objects

```python
import dis


# 1. Disassemble key patterns
def simple_fn(x): return x * 2 + 1

print("--- lambda x: x*2+1 ---")
dis.dis(lambda x: x * 2 + 1)
# LOAD_FAST x | LOAD_CONST 2 | BINARY_OP * | LOAD_CONST 1 | BINARY_OP + | RETURN_VALUE

print("\n--- try/except ---")
def with_try():
    try: return 1 / 0
    except ZeroDivisionError: return 0
dis.dis(with_try)
# Shows PUSH_EXC_INFO, CHECK_EXC_MATCH, POP_EXCEPT — the exception table overhead

# List comprehension compiles to a separate nested code object
print("\n--- [x**2 for x in range(10)] ---")
dis.dis(compile("[x**2 for x in range(10)]", "<lc>", "exec"))
# The comprehension body is in a <listcomp> code object within the outer code


# 2. Code object attributes
code = simple_fn.__code__
print(f"\nco_varnames:   {code.co_varnames}")   # ('x',) — local variable names
print(f"co_freevars:   {code.co_freevars}")    # () — no closure vars
print(f"co_consts:     {code.co_consts}")      # (None, 2, 1)
print(f"co_argcount:   {code.co_argcount}")    # 1
print(f"co_stacksize:  {code.co_stacksize}")   # max operand stack depth needed

# Closure vars
def outer():
    captured = 42
    def inner(): return captured
    return inner

inner = outer()
print(f"co_freevars  (inner): {inner.__code__.co_freevars}")  # ('captured',)
print(f"co_cellvars  (outer): {outer.__code__.co_cellvars}")  # ('captured',)
# co_cellvars: vars in outer that are captured by an inner function
# co_freevars: vars in inner that come from an enclosing scope


# 3. find_globals_used
def find_globals_used(fn) -> set:
    return {
        instr.argval
        for instr in dis.get_instructions(fn)
        if instr.opname in ("LOAD_GLOBAL", "STORE_GLOBAL")
    }

def uses_print_and_len(items):
    print(len(items))

print(f"\nglobals used: {find_globals_used(uses_print_and_len)}")   # {'print', 'len'}


# 4. find_string_literals
def find_string_literals(fn) -> list:
    return [c for c in fn.__code__.co_consts if isinstance(c, str) and c]

def with_strings():
    x = "hello"
    y = "world"
    return x + y

print(f"string literals: {find_string_literals(with_strings)}")   # ['hello', 'world']


# 5. Modify a constant via code.replace() — immutable, but replaceable
def greet():
    return "Hello, World!"

print(f"original: {greet()}")

new_consts = tuple(
    "Goodbye, World!" if c == "Hello, World!" else c
    for c in greet.__code__.co_consts
)
greet.__code__ = greet.__code__.replace(co_consts=new_consts)
print(f"modified: {greet()}")   # Goodbye, World!

# Direct assignment to co_consts fails:
# greet.__code__.co_consts = ("new",)  → AttributeError: readonly attribute


# 6. Stack effects per instruction
print("\nStack effects for simple_fn:")
for instr in dis.get_instructions(simple_fn):
    try:
        effect = dis.stack_effect(instr.opcode, instr.arg if instr.arg is not None else 0)
        print(f"  {instr.opname:25s} effect={effect:+d}")
    except ValueError:
        print(f"  {instr.opname:25s} effect=N/A")
```

---

## Exercise 10 — `sys._getframe`, `traceback`, Frame Objects

```python
import sys, gc, traceback as tb_mod, functools, inspect, time


# 1. caller_info
def caller_info(depth: int = 1) -> dict:
    # depth=0 → this function's frame; depth=1 → caller's frame
    frame = sys._getframe(depth + 1)
    return {
        "function": frame.f_code.co_name,
        "filename": frame.f_code.co_filename.split("/")[-1],
        "lineno":   frame.f_lineno,
        "locals":   dict(frame.f_locals),
    }

def example():
    info = caller_info(0)   # 0 = example's own frame
    print(f"  caller_info: fn={info['function']}, line={info['lineno']}")

example()


# 2. Manual stack walk using frame.f_back
def print_stack_manually():
    frame = sys._getframe(0)
    depth = 0
    while frame is not None:
        print(f"  {'  ' * depth}{frame.f_code.co_name} "
              f"@ {frame.f_code.co_filename.split('/')[-1]}:{frame.f_lineno}")
        frame = frame.f_back
        depth += 1

def level_a():
    def level_b():
        print_stack_manually()
    level_b()

level_a()


# 3. Frame reference memory leak — release promptly
def get_frame_and_release():
    frame = sys._getframe(0)
    info = {"name": frame.f_code.co_name, "lineno": frame.f_lineno}
    del frame   # release immediately — locals in the frame are freed
    gc.collect()
    return info

print(get_frame_and_release())
# Holding frame objects alive longer than necessary keeps ALL their locals in memory


# 4. where() — human-readable current stack
def where():
    stack = tb_mod.extract_stack()
    print("  where() call stack:")
    for frame in stack[:-1]:   # omit where() itself
        print(f"    {frame.name} @ {frame.filename.split('/')[-1]}:{frame.lineno}")

def deep_a():
    def deep_b():
        where()
    deep_b()

deep_a()


# 5. TracebackException for full exception details
def structured_traceback(exc: Exception) -> dict:
    te = tb_mod.TracebackException.from_exception(exc)
    frames = [
        {
            "file":     fs.filename.split("/")[-1],
            "line":     fs.lineno,
            "function": fs.name,
            "text":     fs.line,
        }
        for fs in te.stack
    ]
    result = {
        "type":    te.exc_type.__name__,
        "message": "".join(te.format_exception_only()).strip(),
        "frames":  frames,
        "cause":   None,
    }
    if te.__cause__:
        result["cause"] = {
            "type":    te.__cause__.exc_type.__name__,
            "message": "".join(te.__cause__.format_exception_only()).strip(),
        }
    return result

try:
    try:
        x = 1 / 0
    except ZeroDivisionError as e:
        raise ValueError("Calculation error") from e
except ValueError as exc:
    stb = structured_traceback(exc)
    print(f"  type:    {stb['type']}")
    print(f"  message: {stb['message']}")
    print(f"  frames:  {len(stb['frames'])}")
    print(f"  cause:   {stb['cause']}")


# 7. @profile_calls using frame depth for indentation
def profile_calls(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        depth = len(inspect.stack())
        indent = "  " * depth
        try:
            bound = inspect.signature(fn).bind(*args, **kwargs)
            bound.apply_defaults()
            print(f"{indent}>> {fn.__name__}({dict(bound.arguments)})")
        except Exception:
            print(f"{indent}>> {fn.__name__}({args})")
        t0 = time.perf_counter()
        result = fn(*args, **kwargs)
        ms = (time.perf_counter() - t0) * 1000
        print(f"{indent}<< {fn.__name__} = {result!r} ({ms:.2f}ms)")
        return result
    return wrapper

@profile_calls
def add(a, b): return a + b

@profile_calls
def multiply(a, b): return a * b

add(2, 3)
multiply(4, 5)


# 8. f_locals is a snapshot copy
def show_flocals_snapshot():
    x = 10
    frame = sys._getframe(0)
    frame.f_locals["x"] = 999   # modifying the SNAPSHOT
    print(f"  x after f_locals modification: {x}")   # still 10
    # f_locals returns a fresh dict snapshot each time it is accessed.
    # Writes to it do not propagate back to fast locals in CPython.
    # ctypes.pythonapi.PyFrame_LocalsToFast(frame, ctypes.c_int(0))
    # can force a write-back, but it is an implementation detail and
    # unsafe outside of debugging/profiling tools.

show_flocals_snapshot()
```

---

## Bonus — `Forge`

```python
import sys, ast, dis, gc, time, types, inspect, functools, traceback as tb_mod
from contextlib import contextmanager


# Singleton metaclass (from Metaclasses module)
class SingletonMeta(type):
    _instances = {}
    def __call__(cls, *a, **kw):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*a, **kw)
        return cls._instances[cls]


class Forge(metaclass=SingletonMeta):
    registry = {}
    _patches = []   # [(target, attr, original)]

    # ── inspect_fn ──────────────────────────────────────────────────
    def inspect_fn(self, obj) -> dict:
        sig = inspect.signature(obj) if callable(obj) else None
        params = []
        if sig:
            for name, p in sig.parameters.items():
                params.append({
                    "name":       name,
                    "annotation": str(p.annotation) if p.annotation is not p.empty else None,
                    "default":    None if p.default is p.empty else p.default,
                    "kind":       p.kind.name,
                })
        return {
            "name":     getattr(obj, "__name__", repr(obj)),
            "doc":      inspect.getdoc(obj),
            "params":   params,
            "bytecode": self.bytecode_summary(obj) if callable(obj) else {},
            "members":  {
                "callables": [n for n in dir(obj) if callable(getattr(obj, n, None))
                              and not n.startswith("__")],
                "dunders":   [n for n in dir(obj) if n.startswith("__")],
            }
        }

    # ── patch ────────────────────────────────────────────────────────
    @contextmanager
    def patch(self, target, attr_name, new_value):
        original = getattr(target, attr_name, None)
        self._patches.append((target, attr_name, original))
        setattr(target, attr_name, new_value)
        try:
            yield
        finally:
            setattr(target, attr_name, original)
            if self._patches and self._patches[-1][0] is target:
                self._patches.pop()

    def restore_all(self):
        for target, attr, original in reversed(self._patches):
            setattr(target, attr, original)
        self._patches.clear()

    # ── eval_safe ────────────────────────────────────────────────────
    def eval_safe(self, expr: str, context: dict,
                  allowed_nodes=None) -> object:
        _allowed = allowed_nodes or {
            ast.Expression, ast.BinOp, ast.UnaryOp, ast.Compare,
            ast.BoolOp, ast.Constant, ast.Name, ast.Load,
            ast.Add, ast.Sub, ast.Mult, ast.Div, ast.Mod, ast.Pow,
            ast.USub, ast.UAdd, ast.And, ast.Or, ast.Not,
            ast.Eq, ast.NotEq, ast.Lt, ast.LtE, ast.Gt, ast.GtE,
        }
        tree = ast.parse(expr, mode="eval")
        for node in ast.walk(tree):
            if type(node) not in _allowed:
                raise ValueError(f"Disallowed AST node: {type(node).__name__}")
        return eval(compile(tree, "<forge_safe>", "eval"),
                    {"__builtins__": {}}, context)

    # ── make_class ───────────────────────────────────────────────────
    def make_class(self, name: str, fields: dict,
                   methods: dict = None, validators: dict = None):
        methods    = methods or {}
        validators = validators or {}
        fnames     = list(fields.keys())

        # Build __init__ with an explicit signature via exec
        params      = ", ".join(fnames)
        assignments = "\n    ".join(f"self.{f} = {f}" for f in fnames)
        src = f"def __init__(self, {params}):\n    {assignments}\n    self._validate()"
        ns = {}
        exec(src, ns)
        generated_init = ns["__init__"]

        def _validate(self):
            errors = []
            for fname, validator in validators.items():
                val = getattr(self, fname, None)
                if not validator(val):
                    errors.append(f"  {fname}: validation failed for {val!r}")
            if errors:
                raise ValueError("Validation failed:\n" + "\n".join(errors))

        def __repr__(self):
            parts = ", ".join(f"{f}={getattr(self, f)!r}" for f in fnames)
            return f"{name}({parts})"

        namespace = {
            "__init__":  generated_init,
            "_validate": _validate,
            "__repr__":  __repr__,
        }
        namespace.update(methods)
        cls = type(name, (object,), namespace)
        self.registry[name] = cls
        return cls

    # ── trace ────────────────────────────────────────────────────────
    def trace(self, fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            depth  = len(inspect.stack())
            indent = "  " * depth
            try:
                bound = inspect.signature(fn).bind(*args, **kwargs)
                bound.apply_defaults()
                print(f"{indent}>> {fn.__name__}({dict(bound.arguments)})")
            except Exception:
                print(f"{indent}>> {fn.__name__}({args})")
            t0     = time.perf_counter()
            result = fn(*args, **kwargs)
            ms     = (time.perf_counter() - t0) * 1000
            print(f"{indent}<< {fn.__name__} = {result!r} ({ms:.2f}ms)")
            return result
        return wrapper

    # ── bytecode_summary ─────────────────────────────────────────────
    def bytecode_summary(self, fn) -> dict:
        try:
            instructions = list(dis.get_instructions(fn))
        except TypeError:
            return {}
        opnames = {i.opname for i in instructions}
        return {
            "instructions_count":      len(instructions),
            "globals_used":            {i.argval for i in instructions
                                        if i.opname == "LOAD_GLOBAL"},
            "string_literals":         [c for c in fn.__code__.co_consts
                                        if isinstance(c, str) and c],
            "has_loops":               any(op in opnames
                                           for op in ("FOR_ITER", "GET_ITER")),
            "has_exception_handlers":  any(op in opnames
                                           for op in ("PUSH_EXC_INFO", "SETUP_FINALLY")),
            "constants":               list(fn.__code__.co_consts),
        }


# ── Demo ─────────────────────────────────────────────────────────────

forge = Forge()
assert forge is Forge()   # singleton

@forge.trace
def process(data: list, threshold: float = 0.5) -> list:
    """Filter data above threshold."""
    return [x for x in data if x > threshold]

process([0.3, 0.6, 0.9, 0.1], threshold=0.5)

report = forge.inspect_fn(process)
print(f"\ninspect params: {[p['name'] for p in report['params']]}")
print(f"bytecode has_loops: {report['bytecode']['has_loops']}")

SafeCalc = forge.make_class(
    "SafeCalc",
    fields={"value": int, "label": str},
    methods={"double": lambda self: self.value * 2},
    validators={"value": lambda v: v is not None and v >= 0},
)
obj = SafeCalc(value=10, label="test")
print(f"\n{repr(obj)}")
print(f"double: {obj.double()}")

with forge.patch(SafeCalc, "double", lambda self: self.value * 3):
    print(f"patched double: {obj.double()}")   # 30

print(f"restored double: {obj.double()}")      # 20

print(f"\neval_safe: {forge.eval_safe('x * 2 + y', {'x': 5, 'y': 3})}")  # 13

try:
    forge.eval_safe("len(x)", {"x": [1, 2, 3]})
except ValueError as e:
    print(f"call blocked: {e}")

print(f"\nForge.registry: {list(forge.registry.keys())}")
```
