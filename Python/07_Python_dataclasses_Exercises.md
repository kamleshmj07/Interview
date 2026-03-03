# Python Dataclasses — Practical Exercises for Senior Developers

---

### Exercise 1 — `field()` Customisation

**Context:** A financial instrument catalogue where fields carry complex defaults, some must be excluded from repr or comparison, and metadata carries schema information used by downstream serialisers.

**Tasks:**
1. Define a `Bond` dataclass with these fields and their exact constraints:
   - `isin: str` — no default
   - `coupon_rate: float` — raw default of `0.05`
   - `maturity_date: date` — `default_factory=date.today`
   - `tags: list[str]` — `default_factory=list`; first demonstrate the `ValueError` you get if you write `tags: list = []` directly
   - `cusip: str | None` — `field(default=None, metadata={"description": "CUSIP identifier", "max_length": 9})`
   - `notes: str` — `field(repr=False)` so it is present on the instance but absent from `repr()` output
   - `price: float` — `field(compare=False)` so two bonds with the same `isin` compare equal regardless of `price`
   - `_internal_id: str` — `field(init=False, repr=False, compare=False)`, set to `str(uuid.uuid4())` in `__post_init__`
2. Access `Bond.__dataclass_fields__["cusip"].metadata` and print the dict.
3. Show that `default=` and `default_factory=` are mutually exclusive — demonstrate the error when both are specified.
4. Prove that `notes` is in `bond.__dict__` but absent from `repr(bond)`.
5. Prove that two `Bond` instances with the same `isin` but different `price` are considered equal.
6. Explain (in comments) the three-way `hash` behaviour: `hash=None` (default), `hash=False`, and `hash=True` — when would you explicitly set each?

---

### Exercise 2 — `frozen=True` for Immutability

**Context:** An event-sourcing system where domain events must be immutable once created — any mutation attempt must be a loud error, not a silent corruption.

**Tasks:**
1. Define `@dataclass(frozen=True) class DomainEvent` with `event_id: UUID`, `aggregate_id: str`, `event_type: str`, `occurred_at: datetime`. Attempt `event.event_type = "modified"` and show the `FrozenInstanceError`.
2. Show that `frozen=True` makes instances hashable (assuming all fields are hashable) — place a frozen instance in a `set` and use it as a dict key.
3. Demonstrate the **shallow-freeze gotcha**: the frozen dataclass blocks reassigning `payload: dict`, but `event.payload["hacked"] = True` succeeds because the dict is mutable. Show the fix using `field(default_factory=lambda: MappingProxyType({}))`.
4. Implement `@dataclass(frozen=True) class Vector2D(x: float, y: float)` with a computed `magnitude: float = field(init=False, compare=False)` field. Show that `__post_init__` must use `object.__setattr__(self, "magnitude", ...)` to set it. Implement `normalised()` returning a new `Vector2D` via `replace()`.
5. Show that inheriting a non-frozen dataclass from a frozen one raises `TypeError` — demonstrate both the error and why the reverse (frozen child of non-frozen parent) also fails.
6. Confirm that `@dataclass(frozen=True)` survives `pickle` round-trips and `copy.deepcopy()` — frozen is a runtime invariant, not a serialisation constraint.

---

### Exercise 3 — `__post_init__()` for Validation & Custom Logic

**Context:** A trade execution system where order objects must self-validate on construction, derive computed fields, and enforce business rules that span multiple fields.

**Tasks:**
1. Define `@dataclass class TradeOrder` with `symbol: str`, `side: Literal["BUY", "SELL"]`, `quantity: int`, `price: float`, `order_type: str`. In `__post_init__`:
   - Validate `quantity > 0` and `price >= 0`
   - Raise `ValueError` if `order_type == "MARKET"` and `price != 0.0`
   - Set `notional: float = field(init=False)` to `quantity * price`
   - Set `order_id: str = field(init=False)` to `str(uuid.uuid4())`
2. Confirm that `__post_init__` runs **after** all fields are assigned by printing every field value inside it.
3. Show that `replace()` re-runs `__post_init__` — this is a feature (re-validation) and a footgun (set-once fields). Implement a `created_at` field that is preserved across `replace()` by making it an init-visible field with `default=None` and only setting it in `__post_init__` when it is `None`.
4. Implement `__post_init__` chaining in a two-level hierarchy: `BaseOrder.__post_init__` validates `symbol`, `LimitOrder.__post_init__` calls `super().__post_init__()` then validates `price > 0`.
5. Demonstrate that an exception in `__post_init__` prevents the object from being created — the half-constructed object is garbage collected.

---

### Exercise 4 — Inheritance with Dataclasses

**Context:** A content management system with a base `Asset` type and specialised subtypes — each adds fields while sharing common validation behaviour.

**Tasks:**
1. Define `@dataclass class Asset` with `asset_id: str`, `title: str`, `created_at: datetime = field(default_factory=datetime.now)`, `tags: list = field(default_factory=list)`. Subclass with `@dataclass class ImageAsset(Asset)` adding `width: int = 0`, `height: int = 0`, `format: str = "JPEG"`. Show that `ImageAsset.__dataclass_fields__` contains all seven fields.
2. Demonstrate the **field ordering error**: a base class has `x: int = 0` (with default), the subclass tries to add `y: int` (no default). Show the `TypeError`.
3. Fix the ordering problem three ways:
   - Give the subclass field a default value
   - Use `field(default=None)` with `__post_init__` validation
   - Use the `KW_ONLY` sentinel (Python 3.10+) — show `class Child(Base): _: KW_ONLY; y: int` makes `y` keyword-only and sidesteps the ordering constraint
4. Build a three-level chain: `Asset → MediaAsset(Asset) → VideoAsset(MediaAsset)`. Apply `@dataclass` to every level. Chain `__post_init__` correctly with `super().__post_init__()` at each level. Show `VideoAsset.__dataclass_fields__` contains all fields from all three levels.
5. Show what breaks when `@dataclass` is missing from the intermediate class — demonstrate that `ChildOfUndecorated.__dataclass_fields__` lacks the middle class's fields.
6. Implement a frozen inheritance chain `@dataclass(frozen=True) class FrozenBase → FrozenChild` and show it works. Then show `TypeError` when mixing frozen and non-frozen.

---

### Exercise 5 — `asdict()`, `astuple()`, `make_dataclass()`

**Context:** A data pipeline that serialises nested dataclass graphs for JSON export, converts to tuples for database INSERT, and dynamically creates dataclasses from user-supplied schemas.

**Tasks:**
1. Build a nested model: `Address → Customer(billing: Address, shipping: Address)`, `Product`, `LineItem(product: Product, quantity: int)`, `Order(customer: Customer, items: list[LineItem])`. Call `asdict(order)` and show the recursive dict structure — including the `list[LineItem]` converting to a `list[dict]`.
2. Show that `asdict` deep-copies — mutating a key in the returned dict does not affect the original dataclass.
3. Call `astuple(order)` and show the nested tuple structure. Explain the primary use case: `cursor.execute(INSERT_SQL, astuple(record))`.
4. Show that `asdict` passes non-dataclass types through unchanged — a `datetime` field becomes a `datetime` object, not a string. Implement `to_json_dict(dc) -> dict` that post-processes the output, converting `datetime` → ISO string, `UUID` → str, `Decimal` → float.
5. Use `make_dataclass("DynClass", [("name", str), ("age", int), ("score", float)])` to create a class dynamically. Show it has a working `__init__`, `__repr__`, and `__eq__`.
6. Use `make_dataclass` with the `bases=` parameter to subclass an existing dataclass, and the `namespace=` parameter to inject a custom method.

---

### Exercise 6 — `replace()` for Copy-and-Update

**Context:** An immutable configuration system and a game state manager where "mutations" are expressed as new objects with specific fields changed.

**Tasks:**
1. Use `replace(game_state, score=100)` to create an updated copy — show the original is unchanged.
2. Demonstrate the **shallow copy problem**: the `players: list` field is shared between the original and the copy. Show how mutating `copy.players` also mutates `original.players`.
3. Fix it by passing an explicit copy: `replace(state, players=list(state.players))`.
4. Implement `evolve(dc, **changes)` — a thin wrapper around `replace()` that raises `TypeError` with a descriptive message if any keyword argument is not a valid field name.
5. Show that `replace()` re-runs `__post_init__`. Implement a `created_at: datetime = None` field (init-visible) that is only set when `None` — this ensures `replace()` carries the original timestamp through automatically.
6. Demonstrate `replace()` on a `frozen=True` dataclass — this is its primary use case. Confirm no `FrozenInstanceError` is raised.

---

### Exercise 7 — `eq=False`, `order=True`, Custom Comparisons

**Context:** A priority queue for a job scheduler, and a leaderboard system where ordering and equality semantics must be precisely controlled.

**Tasks:**
1. Define `@dataclass(order=True) class Job` with `priority: int`, `job_id: str`, `payload: str = field(compare=False)`. Show that `sorted(jobs)` and `heapq.heappush` work correctly, ordering by `priority` then by `job_id` as tiebreaker.
2. Prove the **comparison tuple semantics**: `order=True` compares field-by-field in declaration order, identically to comparing tuples — demonstrate two jobs with equal `priority` being further sorted by `job_id`.
3. Show `field(compare=False)` excluding a field from both `==` and ordering — demonstrate that two jobs with the same `priority` and `job_id` but different `payload` are considered equal.
4. Use `@dataclass(eq=False)` to suppress auto-generated `__eq__`, then define a custom `__eq__` that compares only `value`. Show that manually defining `__eq__` makes the class unhashable (Python sets `__hash__ = None`). Fix this by also specifying `unsafe_hash=True`.
5. Show that `@dataclass(order=True, eq=False)` raises `ValueError` at class definition time — explain why: `order=True` requires `eq=True` to generate internally consistent comparison methods.
6. Demonstrate `@dataclass(eq=True, order=False)` (the default) with a manually defined `__lt__` — show that `sorted()` uses it correctly.

---

### Exercise 8 — Nested Dataclasses

**Context:** An e-commerce order model with a rich nested domain graph — orders contain line items, each with a product, attached to a customer with billing and shipping addresses.

**Tasks:**
1. Build the full model: `Address`, `Customer(billing: Address, shipping: Address)`, `Product(unit_price: Decimal)`, `LineItem(product: Product, quantity: int, discount: float = 0.0)`, `Order(customer: Customer, items: list[LineItem])`. Print a deep `repr(order)`.
2. Call `asdict(order)` and show the full recursive dict, including `list[LineItem]` becoming `list[dict]`.
3. Implement `total_price(order) -> Decimal` summing `product.unit_price * quantity * (1 - discount)` across all line items.
4. Use `replace()` to update a `LineItem`'s quantity, then rebuild the `Order` with the updated item list — demonstrate both the shallow-copy bug and the explicit-copy fix.
5. Show the **partial-freeze gotcha**: a `@dataclass(frozen=True)` outer class can still be mutated through a non-frozen nested field — demonstrate, then describe the correct fix (freeze all layers).
6. Implement `Order.from_dict(data: dict) -> Order` as a `@classmethod` that reconstructs the full nested graph from the dict produced by `asdict()` — handle all nested types recursively.

---

### Exercise 9 — `InitVar` for Non-Field Constructor Params

**Context:** A database model system where construction requires a connection object or a raw config dict that bootstraps the instance but must not be stored on it.

**Tasks:**
1. Define `@dataclass class DBRecord` with `table_name: str` and `connection: InitVar[Connection]`. In `__post_init__`, use the connection to set `schema: str = field(init=False)` and `full_path: str = field(init=False)`. Confirm with `DBRecord.__dataclass_fields__` that `connection` is present but tagged as `_FIELD_INITVAR` — and confirm it is **absent** from `asdict()` output.
2. Define `@dataclass class Config` with `raw_config: InitVar[dict]`, parsing it in `__post_init__` to set `host: str`, `port: int`, `debug: bool` — all `field(init=False)`. Show the clean call: `Config(raw_config={"host": "db", "port": 5432, "debug": False})`.
3. Make the `InitVar` optional by giving it `= None` as a default — `enricher: InitVar[dict | None] = None`. In `__post_init__`, skip enrichment when it is `None`.
4. Show that `InitVar` values do **not** appear in `asdict()`, `repr()`, or `astuple()` output — confirm each.
5. Demonstrate `InitVar` in a two-level inheritance chain: `Base` and `Child` each have their own `InitVar`. Show the correct `__post_init__` signature for `Child` and the correct `super().__post_init__(base_initvar)` call.
6. Implement a `ServerConfig.from_env()` classmethod as an alternative to `InitVar` — compare the two approaches in comments and state when each is preferable.

---

### Exercise 10 — `field(init=False)` for Computed Attributes

**Context:** A metrics aggregation system where several fields are always derived from other fields — they must never be set by callers, but must be present, repred, and compared normally.

**Tasks:**
1. Define `@dataclass class SalesMetrics` with init fields `revenue: float`, `cost: float`, `units_sold: int`, and computed `field(init=False)` fields `gross_profit`, `margin_pct`, `profit_per_unit` — all derived in `__post_init__`.
2. Show that `field(init=False)` fields appear in `__repr__`, `==`, and `asdict()` — they are full members of the dataclass, just not constructor parameters.
3. Demonstrate the constraint: a `field(init=False)` field **must** declare `default=` or `default_factory=`, or Python raises `TypeError` when the class is defined. Show both the broken and fixed versions.
4. Implement a `Timestamped` base dataclass with `created_at` and `updated_at` both as `field(init=False, default_factory=datetime.now)`. Mix it into a `ProductFixed` class. Show the field ordering constraint: because `Timestamped` uses defaults, any subclass fields that need to be positional must also have defaults.
5. Show the `frozen=True` + `field(init=False)` interaction: you cannot do `self.field = value` in `__post_init__` of a frozen class. Show the `FrozenInstanceError`, then show the fix using `object.__setattr__(self, "field", value)`.
6. Implement a `_cache: dict = field(init=False, repr=False, compare=False, default_factory=dict)` for memoising an expensive computation. Confirm `_cache` is absent from `repr()` and invisible to `==`, but note it **is** included in `asdict()` output — and show how to exclude it by also setting `field(hash=False)` and post-filtering `asdict`.

---

## Bonus Challenge — Bring It All Together

Design a **typed, immutable, self-validating domain model** for a bond trading platform.

**Requirements:**

- `@dataclass(frozen=True) class Money` — `amount: Decimal`, `currency: str`. `__post_init__` rounds `amount` to 2 d.p. and validates `currency` is a 3-letter alpha code. Implement `__add__`, `__mul__`, `__lt__`, and `to_dict()`.
- `@dataclass(frozen=True) class BondSpec` — use `field(init=False)` for `country_code` and `issuer_code` derived from `isin` in `__post_init__` via `object.__setattr__`.
- `@dataclass class TradeTicket` — inherits from `Auditable` which injects `created_at: datetime = None`, `updated_at: datetime = None`, and `version: int = 0` as init-visible fields. `__post_init__` on `TradeTicket` increments `version` — because these are init-visible, `replace()` carries the current `version` through and `__post_init__` increments it correctly.
- An `amend(ticket, **changes)` function that returns `replace(ticket, **changes)` with `updated_at` refreshed.
- Full `asdict()` / `from_dict()` round-trip for every type.

```python
spec = BondSpec(
    isin="US912810TD00",
    coupon_rate=0.0425,
    maturity=date(2053, 2, 15),
    notional=Money(Decimal("1000000"), "USD"),
)
ticket = TradeTicket(bond=spec, side="BUY", quantity=10)
print(ticket.version)          # 1

amended = amend(ticket, quantity=20)
print(amended.quantity)        # 20
print(amended.version)         # 2
print(ticket.quantity)         # 10  — original unchanged
print(amended is not ticket)   # True
```
