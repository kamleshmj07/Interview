# Python Metaprogramming — Practical Exercises for Senior Developers

> **Scope:** Covers only topics not addressed in previous exercise sets.
> Excluded (already covered): Metaclasses, Decorators, ABCs, `type()` class
> creation, Descriptor basics as ORM fields, Function factories.
> **Descriptors** receive a full dedicated treatment here.

---

## INTROSPECTION

---

### Exercise 1 — `inspect` Module Deep Dive

**Context:** You're building a self-documenting API framework. At startup it scans all registered handler functions and auto-generates an OpenAPI-style schema — parameter names, types, defaults, docstrings, source locations — purely from introspection.

**Tasks:**
1. Given a function with full annotations, defaults, `*args`, `**kwargs`, and a multi-section docstring, use `inspect.signature()` to extract: parameter names, kinds (`POSITIONAL_ONLY`, `KEYWORD_ONLY`, `VAR_POSITIONAL`, etc.), annotations, and default values. Reconstruct the signature string manually from this data.
2. Use `inspect.getmembers(cls, predicate)` to categorise all members of a class into: instance methods, properties, classmethods, staticmethods, dunder methods, and plain attributes — using `inspect.ismethod`, `inspect.isfunction`, and `inspect.isdatadescriptor` as predicates.
3. Use `inspect.getsource()` and `inspect.getsourcelines()` to extract and display the source of a live function. Show it raises `OSError`/`TypeError` for built-in C functions.
4. Show the difference between `fn.__doc__` and `inspect.getdoc(fn)` — `getdoc` strips indentation while `__doc__` preserves raw whitespace. Show this with a function whose docstring is indented.
5. Use `inspect.stack()` inside a nested call chain `a() → b() → c()` to print a formatted call stack showing function name, filename, and line number at each frame.
6. Implement `function_schema(fn) -> dict` returning `{"name", "doc", "params": [{name, annotation, default, kind}], "return_annotation", "source_file", "source_line"}` using only `inspect` APIs — no manual `__doc__` or `__annotations__` string parsing.

**Constraint:** No manual string parsing of `__doc__` or `__annotations__` — use `inspect` APIs throughout.

---

### Exercise 2 — `dir()`, `vars()`, and `callable()`

**Context:** A live REPL-style object explorer for a debugging console — must introspect any object and categorise everything it exposes.

**Tasks:**
1. Show the three-way difference between `dir(obj)`, `vars(obj)`, and `obj.__dict__`: when each works, what each omits, and a case where `vars()` raises `TypeError` — specifically on a `__slots__`-only class and on a built-in.
2. Implement `explore(obj) -> dict` returning four keys:
   - `"callables"`: all callable non-dunder attributes
   - `"properties"`: all `property` objects found anywhere in the MRO
   - `"data"`: all non-callable, non-dunder instance attributes
   - `"dunders"`: all dunder attribute names
3. Demonstrate that `dir()` walks the full MRO — show that `dir(grandchild_instance)` includes attributes from every ancestor class, while `vars(GrandChild)` only shows attributes defined directly on `GrandChild`.
4. Use `callable()` to confirm all of these return `True`: a plain function, a lambda, a class itself, an instance with `__call__`, a built-in function, and a bound method.
5. Implement `attribute_origin(obj, name) -> type` — returns the class in `type(obj).__mro__` where the attribute is **actually defined** (not merely inherited), using `vars()` on each class in the MRO chain.
6. Customise `dir()` output by overriding `__dir__` — implement a class where `dir()` hides `_private` attributes and adds virtual attribute names from an internal `_virtual_attrs` list.

---

## DYNAMIC ATTRIBUTE ACCESS

---

### Exercise 3 — `getattr` / `setattr` / `delattr` / `hasattr`

**Context:** A dynamic proxy system for a plugin framework — the proxy forwards attribute access to one of several backends based on routing rules, with fallback chains, access logging, and runtime attribute injection.

**Tasks:**
1. Implement `DynamicProxy(target)` — all attribute access on the proxy forwards to `target` via `getattr`. `AttributeError` is caught and re-raised with a more descriptive message. Use `__getattr__` (not `__getattribute__`) and explain precisely when each fires.
2. Implement `RoutingProxy(routes: dict)` — `routes` maps `fnmatch`-style patterns to backend objects. `proxy.db_users` routes to the database backend, `proxy.cache_users` to the cache backend. Use `fnmatch.fnmatch` inside `__getattr__`.
3. Implement `apply_config(obj, config: dict)` — uses `setattr` to bulk-apply a config dict; skips keys starting with `_` and silently skips read-only attributes by catching `AttributeError`.
4. Implement `safe_call(obj, method_name, *args, **kwargs)` — calls the method only if it exists (using `hasattr`), returns `None` otherwise. Explain that `hasattr(obj, name)` is equivalent to `try: getattr(obj, name); return True except AttributeError: return False`.
5. Implement `deep_getattr(obj, dotted_path: str, default=MISSING)` — supports `deep_getattr(config, "database.host.port", default=5432)` by splitting on `.` and chaining `getattr` calls; returns `default` if any segment is missing.
6. Demonstrate the `hasattr` footgun: a `@property` that raises `AttributeError` internally due to a bug is silently swallowed — `hasattr` returns `False` even though the attribute *exists*. Show the bug, then show the safe diagnostic using `inspect.getattr_static`.

---

## DYNAMIC CODE EXECUTION

---

### Exercise 4 — `eval`, `exec`, and `compile`

**Context:** A configuration DSL, a sandboxed expression evaluator for a rules engine, and a code-generation tool that constructs classes at runtime.

**Tasks:**

**`eval`:**
1. Implement `safe_eval(expr: str, context: dict) -> Any` — evaluates a mathematical/logical expression with a restricted namespace: `__builtins__` set to `{}` and only whitelisted names allowed. Show that `eval("__import__('os').system('ls')", {})` is blocked by this restriction.
2. Show that `eval` only accepts **expressions**, not statements — `eval("x = 1")` raises `SyntaxError`. Demonstrate the conceptual boundary between expression and statement.

**`exec`:**
3. Use `exec` to dynamically define a function from a generated string. Capture the resulting function from the `locals` dict passed to `exec` and call it.
4. Implement `make_init(fields: list[str]) -> callable` — uses `exec` to generate an `__init__` with an **explicit** signature `(self, field1, field2, ...)` rather than `(**kwargs)`. This is how `dataclasses` generates `__init__` internally — show `inspect.signature` reflects the generated parameters.
5. Demonstrate the CPython `exec` locals scoping pitfall: `exec("x = 99")` inside a function does **not** modify the enclosing function's `x`. Show the behaviour and explain why (hint: CPython optimises local variable access to fast slots, not dict lookups).

**`compile`:**
6. Use `compile(source, filename, mode)` with modes `"eval"`, `"exec"`, and `"single"`. Show the differences in the returned code objects.
7. Compile a code object once with `compile`, then execute it multiple times with `eval(code_obj, namespace)` — use `timeit` to demonstrate it is measurably faster than re-parsing the string on every call.
8. Use `ast.parse` to inspect the AST before executing — parse an expression, walk the AST to reject any `ast.Call` nodes (disallowing function calls), then compile and eval the sanitised expression.

---

## DESCRIPTORS

---

### Exercise 5 — Data Descriptors (`__get__`, `__set__`, `__delete__`)

**Context:** A validated ORM-style field system — every field enforces constraints at assignment time, supports lazy defaults, and integrates with the instance `__dict__` without interference.

**Tasks:**
1. Implement a `DataDescriptor` base with `__set_name__`, `__get__`, `__set__`, `__delete__`. Prove the lookup priority: **data descriptor > instance `__dict__` > non-data descriptor > class attribute**. Write one class demonstrating each priority level.
2. Implement `Typed(type_)` — a data descriptor that enforces type at `__set__`. Show that without `__set__` you could bypass it by writing directly to `instance.__dict__["name"]` — and that adding `__set__` closes this bypass.
3. Implement `Validated(validator_fn, error_msg)` wrapping any single-argument predicate. Compose `Typed(int)` and `Validated(lambda x: x > 0, "must be positive")` into a `Constrained(*descriptors)` that chains validation through all of them.
4. Implement `Lazy(factory_fn)` — a **non-data descriptor** (no `__set__`) that computes its value on first access and caches it in the instance's `__dict__`, so all subsequent accesses bypass the descriptor entirely. Prove the caching by showing `"area" not in c.__dict__` before first access and `"area" in c.__dict__` after.
5. Implement `Observable(initial)` — a data descriptor that maintains a list of subscriber callbacks. When `__set__` is called with a new value, all subscribers are notified with `(instance, old_value, new_value)`.
6. Show descriptor priority with four concrete classes: `DataDescPriority`, `InstanceDictPriority`, `NonDataDescPriority`, `ClassAttrPriority` — each demonstrating the next lower priority being overridden by the one above.

---

### Exercise 6 — Non-Data Descriptors & `__set_name__`

**Context:** A caching and method-binding system — understanding non-data descriptors is essential for understanding how Python's own functions, `classmethod`, and `staticmethod` work internally.

**Tasks:**
1. Replicate Python's method-binding mechanism from scratch: implement `BoundMethod` and a `Function` descriptor so that `obj.method(arg)` correctly passes `obj` as `self` — without using any Python built-in method machinery.
2. Replicate `@classmethod` as a `ClassMethod` descriptor — `__get__` returns a partial that injects `cls` (the type, not the instance) as the first argument, regardless of whether accessed on the class or an instance.
3. Replicate `@staticmethod` as a `StaticMethod` descriptor — `__get__` returns the raw function unchanged in all cases.
4. Implement `AutoSlug` using `__set_name__` — when a string value is assigned, automatically converts it to a URL-safe slug (lowercase, spaces and special chars → hyphens). Uses `__set_name__` to capture the attribute name for per-instance storage in `instance.__dict__`.
5. Implement `VersionedAttribute` — stores a full history of every value ever assigned. `__get__` returns the current (most recent) value; a `get_history(instance)` method on the descriptor returns the full list.
6. Verify that `classmethod` and `staticmethod` are **non-data** descriptors: `hasattr(classmethod, "__set__")` is `False`. Explain the consequence — instance `__dict__` entries *can* shadow them — and why this is rarely a problem in practice.

---

## MONKEY PATCHING

---

### Exercise 7 — Runtime Code Alteration

**Context:** A testing framework, a hot-fix deployment system, and a third-party library integration layer — all requiring safe, reversible runtime modification of live objects and classes.

**Tasks:**
1. Implement `patch(target, attr_name, new_value)` as a **context manager** — saves the original value, replaces it on entry, restores it on exit (including on exception). Show it working to temporarily replace a method on a class during a test.
2. Add a method to an existing plain-Python class by assigning a function to `ExistingClass.new_method = fn`. Show it works on instances that were created *before* the patch was applied.
3. Demonstrate the **bound method pitfall** when patching an **instance** directly: `instance.method = fn` does not pass `self` automatically. Show the broken behaviour, then fix it using `types.MethodType(fn, instance)`.
4. Implement `MonkeyPatchRegistry` — tracks every applied patch (target, attr, original value). `registry.restore_all()` undoes every patch in reverse order.
5. Demonstrate MRO propagation: patching a method on a **base class** affects all subclasses that do not override it — show with a three-level hierarchy. Then show that a subclass that *does* override the method is unaffected.
6. Show why patching `__dunder__` methods on an **instance** silently fails — Python's special method lookup bypasses `__getattribute__` and reads directly from `type(obj)`. Demonstrate `obj.__len__ = lambda: 99` having no effect on `len(obj)`, then show the correct fix (patch the class, or use a wrapper class).

---

## FUNCTOOLS: PARTIAL & PARTIALMETHOD

---

### Exercise 8 — `functools.partial`, `partialmethod`, and `reduce`

**Context:** A configurable data processing pipeline where operations are pre-parameterised and composed — using `partial` to freeze arguments and `reduce` to chain transformations.

**Tasks:**
1. Use `functools.partial` to create pre-configured callables: `open` with `encoding="utf-8"` and `mode="r"` frozen; `sorted` with `key=str.lower` frozen; a `clamp(value, lo, hi)` specialised as `clamp_to_byte = partial(clamp, lo=0, hi=255)`.
2. Introspect a `partial` object's `.func`, `.args`, and `.keywords` attributes. Show that `partial` does **not** have `__wrapped__` (unlike `functools.wraps`-decorated functions) and explain why.
3. Implement `pipeline(value, *fns)` using `functools.reduce` — threads `value` through each function in sequence. Prove `pipeline(v, f, g, h)` equals `h(g(f(v)))`.
4. Use `functools.partialmethod` to define REST method shorthands on a class: a `_request(self, method, path, **kwargs)` base method with `get = partialmethod(_request, "GET")` and `post = partialmethod(_request, "POST")`.
5. Compare `partial(fn, arg)` vs `lambda: fn(arg)` — both partially apply, but `partial` exposes `.func`, `.args`, `.keywords` for introspection while a lambda is opaque. Show both and explain when each is preferable.
6. Implement `memoize_with_ttl(ttl_seconds)` — returns a decorator that caches results with a time-to-live expiry. Use `partial` to create a family: `cache_1s = memoize_with_ttl(1)`, `cache_60s = memoize_with_ttl(60)`.

---

## BYTECODE MANIPULATION

---

### Exercise 9 — `dis` Module & Code Objects

**Context:** A performance profiler and optimisation tool — understanding bytecode is essential for knowing where CPython spends time and for programmatically analysing or modifying code objects.

**Tasks:**
1. Disassemble the following with `dis.dis()` and explain every instruction for each:
   - A simple `lambda x: x * 2 + 1`
   - A list comprehension `[x**2 for x in range(10)]`
   - A function containing a `try/except` block
   Show that list comprehensions compile to a distinct nested code object.
2. Access a function's code object via `fn.__code__`. Inspect and explain each of these attributes:
   - `co_varnames`, `co_freevars`, `co_cellvars` — locals, closure-captured vars, vars captured by inner functions
   - `co_consts`, `co_names` — constants and global names referenced
   - `co_argcount`, `co_kwonlyargcount`, `co_flags`
   - `co_stacksize`, `co_firstlineno`
3. Use `dis.get_instructions(fn)` to implement `find_globals_used(fn) -> set[str]` — returns every global name the function reads, by filtering for `LOAD_GLOBAL` instructions.
4. Implement `find_string_literals(fn) -> list[str]` — returns all non-empty string constants from `fn.__code__.co_consts`.
5. Show that code objects are **immutable** — you cannot assign to `fn.__code__.co_consts` directly. Then use `fn.__code__.replace(co_consts=new_consts)` (Python 3.8+) to create a modified copy and swap it back onto `fn.__code__` — effectively changing a string constant in a live function without redefining it.
6. Use `dis.stack_effect(opcode, oparg)` to compute the net stack depth change of each instruction in a function — print the instruction name and effect for each.

---

## FRAME INSPECTION

---

### Exercise 10 — `sys._getframe`, `traceback`, and Frame Objects

**Context:** A structured logging library, a profiler, and a debugging tool — all requiring deep access to the call stack at runtime.

**Tasks:**

**Frame access:**
1. Use `sys._getframe(depth)` to implement `caller_info(depth=1) -> dict` returning `{"function", "filename", "lineno", "locals"}` of the calling frame. Verify: `caller_info(0)` is the frame of `caller_info` itself; `caller_info(1)` is whoever called it.
2. Use `frame.f_locals`, `frame.f_globals`, `frame.f_code`, and `frame.f_back` to walk the **entire call stack manually** from the current frame to the outermost frame — print a formatted call stack without using the `traceback` module.
3. Demonstrate the frame reference memory leak risk: holding a frame object keeps all its local variables alive. Implement `get_frame_and_release()` — extract needed data, `del` the frame reference immediately, then use `gc.collect()` to confirm the frame is eligible for collection.

**`traceback` module:**
4. Implement `where()` using `traceback.extract_stack()` and `traceback.format_stack()` — prints the current call stack in a clean human-readable format from inside a deeply nested call chain.
5. Capture a live exception with `sys.exc_info()` and use `traceback.TracebackException` to extract: exception type, message, every frame in the traceback with filename/lineno/function/source line, and the chained `__cause__`.
6. Implement `structured_traceback(exc: Exception) -> dict` — converts a full exception traceback to a JSON-serialisable dict: `{"type", "message", "frames": [{"file", "line", "function", "text"}], "cause": ...}`.

**Advanced:**
7. Implement `@profile_calls` — a decorator that uses `sys._getframe` to count stack depth and logs every function entry/exit with indentation proportional to call depth.
8. Show that `frame.f_locals` is a **snapshot copy** in CPython — modifying it does not change actual local variables. Demonstrate this, then explain the `ctypes.pythonapi.PyFrame_LocalsToFast` mechanism that can write back (with caveats).

---

## Bonus Challenge — Bring It All Together

Design **`Forge`** — a zero-dependency runtime code analysis and patching toolkit.

**Requirements:**

- `Forge` is a **singleton** (using the metaclass pattern from the previous module).
- `forge.inspect_fn(obj) -> dict` — full introspection report using `inspect`, `dis`, `vars()`, `dir()` combining Exercises 1–2.
- `forge.patch(target, attr, value)` — context manager; restores on exit or exception; all patches logged.
- `forge.restore_all()` — undoes every active patch in reverse order.
- `forge.eval_safe(expr, context)` — sandboxed eval using AST inspection to reject disallowed node types (e.g. `ast.Call`) before execution.
- `forge.make_class(name, fields, methods, validators)` — uses `exec` + `type()` to construct a fully-validated class with typed fields, auto-generated `__init__`/`__repr__`, and registers it in `Forge.registry`.
- `forge.trace(fn)` — decorator using frame inspection to log every call with full argument bindings via `inspect.signature().bind()`, call depth, and duration.
- `forge.bytecode_summary(fn) -> dict` — uses `dis` to return `{"instructions_count", "globals_used", "string_literals", "has_loops", "has_exception_handlers", "constants"}`.

```python
forge = Forge()
assert forge is Forge()   # singleton

@forge.trace
def process(data: list, threshold: float = 0.5) -> list:
    """Filter data above threshold."""
    return [x for x in data if x > threshold]

process([0.3, 0.6, 0.9], threshold=0.5)

report = forge.inspect_fn(process)
print(report["params"])                    # [{'name': 'data', ...}, ...]
print(report["bytecode"]["has_loops"])     # True

SafeCalc = forge.make_class(
    "SafeCalc",
    fields={"value": int, "label": str},
    methods={"double": lambda self: self.value * 2},
    validators={"value": lambda v: v is not None and v >= 0},
)
obj = SafeCalc(value=10, label="test")
print(obj.double())   # 20

with forge.patch(SafeCalc, "double", lambda self: self.value * 3):
    print(obj.double())   # 30 — patched
print(obj.double())       # 20 — restored

print(forge.eval_safe("x * 2 + y", {"x": 5, "y": 3}))   # 13
print(forge.registry)     # {'SafeCalc': <class 'SafeCalc'>}
```
