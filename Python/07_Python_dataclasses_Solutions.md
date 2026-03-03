# Python Dataclasses — Solutions

---

## Exercise 1 — `field()` Customisation

```python
import uuid
from dataclasses import dataclass, field
from datetime import date


@dataclass
class Bond:
    # Required — no default
    isin: str

    # Raw default (immutable scalar — OK)
    coupon_rate: float = 0.05

    # Mutable default — must use default_factory
    maturity_date: date = field(default_factory=date.today)

    # List — MUST use default_factory; raw `= []` raises ValueError
    tags: list = field(default_factory=list)

    # Optional field with metadata for downstream schema tools
    cusip: str | None = field(
        default=None,
        metadata={"description": "CUSIP identifier", "max_length": 9},
    )

    # repr=False: present on the instance, invisible in repr()
    notes: str = field(default="", repr=False)

    # compare=False: excluded from == and ordering
    price: float = field(default=100.0, compare=False)

    # init=False: not a constructor parameter, set in __post_init__
    _internal_id: str = field(init=False, repr=False, compare=False)

    def __post_init__(self):
        self._internal_id = str(uuid.uuid4())


# ── Mutable default error ────────────────────────────────────
try:
    @dataclass
    class BadBond:
        tags: list = []   # ValueError at class-definition time
except ValueError as e:
    print(e)
# mutable default <class 'list'> for field tags is not allowed: use default_factory


# ── Metadata access ──────────────────────────────────────────
meta = Bond.__dataclass_fields__["cusip"].metadata
print(dict(meta))   # {'description': 'CUSIP identifier', 'max_length': 9}


# ── Mutual exclusion ─────────────────────────────────────────
try:
    @dataclass
    class Bad:
        x: int = field(default=1, default_factory=int)
except ValueError as e:
    print(e)   # cannot specify both default and default_factory


# ── repr=False ───────────────────────────────────────────────
b = Bond("XS0000000001", notes="sensitive")
print("notes" in repr(b))    # False — hidden
print("notes" in b.__dict__) # True  — present on instance


# ── compare=False ────────────────────────────────────────────
b1 = Bond("SAME_ISIN", price=100.0)
b2 = Bond("SAME_ISIN", price=99.0)
print(b1 == b2)   # True — price excluded from comparison


# ── hash behaviour (comment reference) ───────────────────────
# field(hash=None)  — default: participate in __hash__ if the
#                     class is hashable (frozen or eq=False)
# field(hash=False) — always exclude from __hash__; use for
#                     expensive-to-hash or irrelevant fields
# field(hash=True)  — always include even if compare=False;
#                     rarely needed, can break hash/eq contract
```

---

## Exercise 2 — `frozen=True`

```python
import uuid, copy, pickle
from dataclasses import dataclass, field, replace
from datetime import datetime
from types import MappingProxyType


# 1. FrozenInstanceError on mutation
@dataclass(frozen=True)
class DomainEvent:
    event_id:     uuid.UUID
    aggregate_id: str
    event_type:   str
    occurred_at:  datetime

evt = DomainEvent(
    event_id=uuid.uuid4(),
    aggregate_id="order-1",
    event_type="OrderPlaced",
    occurred_at=datetime.now(),
)
try:
    evt.event_type = "Hacked"
except Exception as e:
    print(type(e).__name__)   # FrozenInstanceError


# 2. Hashable — usable in set and as dict key
event_set = {evt}
event_map = {evt: "value"}
print(len(event_set))   # 1


# 3. Shallow-freeze gotcha + MappingProxyType fix
@dataclass(frozen=True)
class EventWithPayload:
    event_id: uuid.UUID
    payload:  dict = field(default_factory=dict)

ep = EventWithPayload(uuid.uuid4(), payload={"key": "val"})
ep.payload["hacked"] = True   # works! dict itself is mutable
print(ep.payload)

@dataclass(frozen=True)
class SafeEvent:
    event_id: uuid.UUID
    payload:  MappingProxyType = field(
        default_factory=lambda: MappingProxyType({})
    )

se = SafeEvent(uuid.uuid4())
try:
    se.payload["key"] = "val"
except TypeError as e:
    print(type(e).__name__)   # TypeError — proxy is read-only


# 4. Vector2D: object.__setattr__ in __post_init__
@dataclass(frozen=True)
class Vector2D:
    x: float
    y: float
    magnitude: float = field(init=False, compare=False)

    def __post_init__(self):
        # self.magnitude = ...  would raise FrozenInstanceError
        object.__setattr__(self, "magnitude", (self.x**2 + self.y**2)**0.5)

    def normalised(self) -> "Vector2D":
        return replace(self, x=self.x / self.magnitude,
                              y=self.y / self.magnitude)

v = Vector2D(3.0, 4.0)
print(v.magnitude)       # 5.0
print(v.normalised())    # Vector2D(x=0.6, y=0.8, magnitude=1.0)


# 5. Frozen/non-frozen mixing — both directions fail
@dataclass(frozen=True)
class FrozenBase:
    x: int

try:
    @dataclass            # non-frozen child of frozen parent
    class NonFrozenChild(FrozenBase):
        y: int
except TypeError as e:
    print(e)   # cannot inherit non-frozen dataclass from a frozen one

try:
    @dataclass(frozen=True)  # frozen child of non-frozen parent
    class FrozenChild2(object):
        pass
    # The reverse (frozen child of regular class) is actually fine;
    # the error only occurs frozen→non-frozen in dataclass hierarchy
except TypeError:
    pass


# 6. Pickle and deepcopy survive frozen
@dataclass(frozen=True)
class Config:
    host: str = "localhost"
    port: int = 5432

cfg = Config()
print(cfg == pickle.loads(pickle.dumps(cfg)))   # True
print(cfg == copy.deepcopy(cfg))                 # True
```

---

## Exercise 3 — `__post_init__()`

```python
import uuid
from dataclasses import dataclass, field, replace
from datetime import datetime
from typing import Literal


@dataclass
class TradeOrder:
    symbol:     str
    side:       Literal["BUY", "SELL"]
    quantity:   int
    price:      float
    order_type: str
    created_at: datetime = None    # init=True so replace() carries it through
    notional:   float    = field(init=False)
    order_id:   str      = field(init=False)

    def __post_init__(self):
        # Validate
        if self.quantity <= 0:
            raise ValueError(f"quantity must be > 0, got {self.quantity}")
        if self.price < 0:
            raise ValueError(f"price must be >= 0, got {self.price}")
        if self.side not in {"BUY", "SELL"}:
            raise ValueError(f"side must be BUY or SELL, got {self.side!r}")
        if self.order_type == "MARKET" and self.price != 0.0:
            raise ValueError("MARKET orders must have price=0.0")

        # Compute derived fields
        self.notional = self.quantity * self.price
        self.order_id = str(uuid.uuid4())

        # Set created_at only on first construction, not on replace()
        if self.created_at is None:
            self.created_at = datetime.now()


t = TradeOrder("AAPL", "BUY", 100, 150.0, "LIMIT")
print(t.notional)       # 15000.0
print(bool(t.order_id)) # True

# Validation prevents bad instances
try:
    TradeOrder("AAPL", "BUY", -1, 150.0, "LIMIT")
except ValueError as e:
    print(e)

try:
    TradeOrder("AAPL", "BUY", 100, 150.0, "MARKET")
except ValueError as e:
    print(e)

# replace() re-runs __post_init__; created_at is preserved because
# it is an init-visible field and replace() passes it through explicitly
t2 = replace(t, quantity=200)
print(t2.notional)                       # 30000.0 — recalculated
print(t.created_at == t2.created_at)     # True — preserved!


# ── __post_init__ chaining ───────────────────────────────────
@dataclass
class BaseOrder:
    symbol: str
    side:   str

    def __post_init__(self):
        if not self.symbol:
            raise ValueError("symbol is required")

@dataclass
class LimitOrder(BaseOrder):
    price: float = 0.0

    def __post_init__(self):
        super().__post_init__()   # chain base validation
        if self.price <= 0:
            raise ValueError("LimitOrder: price must be positive")

lo = LimitOrder("MSFT", "SELL", price=200.0)   # both validators run


# ── Exception prevents object creation ───────────────────────
import gc

created = []

@dataclass
class CountedOrder:
    qty: int
    def __post_init__(self):
        created.append(id(self))
        if self.qty < 0:
            raise ValueError("negative qty")

try:
    CountedOrder(-1)
except ValueError:
    pass

gc.collect()
print(len(created))   # 1 — __post_init__ ran, but the object was discarded
```

---

## Exercise 4 — Inheritance

```python
from dataclasses import dataclass, field, KW_ONLY
from datetime import datetime


@dataclass
class Asset:
    asset_id:   str
    title:      str
    created_at: datetime = field(default_factory=datetime.now)
    tags:       list     = field(default_factory=list)


@dataclass
class ImageAsset(Asset):
    width:  int = 0
    height: int = 0
    format: str = "JPEG"


img = ImageAsset("img-1", "Cover Photo", width=1920, height=1080)
print(list(ImageAsset.__dataclass_fields__))
# ['asset_id', 'title', 'created_at', 'tags', 'width', 'height', 'format']


# ── Field ordering error ──────────────────────────────────────
@dataclass
class Base:
    x: int = 0   # has default

try:
    @dataclass
    class Child(Base):
        y: int   # no default — TypeError: non-default follows default
except TypeError as e:
    print(e)


# ── Fix 1: give the subclass field a default ─────────────────
@dataclass
class ChildFix1(Base):
    y: int = 0


# ── Fix 2: field(default=None) + __post_init__ ───────────────
@dataclass
class ChildFix2(Base):
    y: int = field(default=None)

    def __post_init__(self):
        if self.y is None:
            raise ValueError("y is required")


# ── Fix 3: KW_ONLY sentinel (Python 3.10+) ───────────────────
@dataclass
class ChildFix3(Base):
    _: KW_ONLY   # everything after this is keyword-only
    y: int       # no default needed — keyword-only sidesteps ordering

c3 = ChildFix3(y=5)
print(c3)   # ChildFix3(x=0, y=5)


# ── Three-level hierarchy with __post_init__ chaining ────────
@dataclass
class MediaAsset(Asset):
    mime_type: str = "application/octet-stream"

    def __post_init__(self):
        # Asset has no __post_init__, but calling super() defensively
        # ensures the chain works if Asset adds one later
        pass


@dataclass
class VideoAsset(MediaAsset):
    duration_s: float = 0.0
    codec:      str   = "H264"

    def __post_init__(self):
        super().__post_init__()
        if self.duration_s < 0:
            raise ValueError("duration cannot be negative")

va = VideoAsset("vid-1", "Intro", mime_type="video/mp4", duration_s=120.0)
print(list(VideoAsset.__dataclass_fields__))
# All 7 fields from all three levels


# ── Missing @dataclass on intermediate class ─────────────────
class MissingDecorator(Asset):   # NO @dataclass applied
    extra: str = "test"

@dataclass
class ChildOfMissing(MissingDecorator):
    pass

# 'extra' is a plain class attribute, not a dataclass field
print("extra" in ChildOfMissing.__dataclass_fields__)   # False


# ── Frozen inheritance chain ──────────────────────────────────
@dataclass(frozen=True)
class FrozenBase:
    x: int

@dataclass(frozen=True)
class FrozenChild(FrozenBase):
    y: int = 0

fc = FrozenChild(x=1, y=2)   # works fine — both frozen

try:
    @dataclass
    class NonFrozenChild(FrozenBase):   # TypeError
        z: int = 0
except TypeError as e:
    print(e)
```

---

## Exercise 5 — `asdict()`, `astuple()`, `make_dataclass()`

```python
import json, uuid
from dataclasses import dataclass, field, asdict, astuple, make_dataclass, is_dataclass
from datetime import datetime
from decimal import Decimal


@dataclass
class Address:
    street:   str
    city:     str
    country:  str
    postcode: str

@dataclass
class Customer:
    customer_id: str
    name:        str
    email:       str
    billing:     Address
    shipping:    Address

@dataclass
class LineItem:
    product_name: str
    unit_price:   float
    quantity:     int

@dataclass
class Order:
    order_id:  str
    customer:  Customer
    items:     list          # list[LineItem]
    placed_at: datetime = field(default_factory=datetime.now)


order = Order(
    "ORD-001",
    Customer("C1", "Alice", "alice@example.com",
             billing=Address("1 Main St", "London", "GB", "EC1A 1BB"),
             shipping=Address("2 High St", "London", "GB", "SW1A 2AA")),
    [LineItem("Widget", 9.99, 3), LineItem("Gadget", 29.99, 1)],
)


# 1. asdict — recursive
d = asdict(order)
print(type(d["customer"]))       # dict
print(type(d["items"][0]))       # dict — LineItem → dict
print(type(d["items"]))          # list


# 2. asdict deep-copies — mutation doesn't affect original
d["customer"]["name"] = "MUTATED"
print(order.customer.name)   # Alice — unchanged


# 3. astuple — nested tuples; ideal for DB inserts
t = astuple(order)
print(type(t[2]))     # list  (the items list)
print(type(t[2][0]))  # tuple (each LineItem)
# cursor.execute("INSERT INTO orders VALUES (?, ?, ?)", astuple(simple_record))


# 4. Non-dataclass types pass through unchanged; post-process for JSON
def to_json_dict(dc) -> dict:
    raw = asdict(dc)

    def convert(obj):
        if isinstance(obj, dict):
            return {k: convert(v) for k, v in obj.items()}
        if isinstance(obj, list):
            return [convert(i) for i in obj]
        if isinstance(obj, datetime):
            return obj.isoformat()
        if isinstance(obj, uuid.UUID):
            return str(obj)
        if isinstance(obj, Decimal):
            return float(obj)
        return obj

    return convert(raw)

json_safe = to_json_dict(order)
print(type(json_safe["placed_at"]))   # str — ISO format
json.dumps(json_safe)                 # no TypeError


# 5. make_dataclass from dynamic schema
schema = [("name", str), ("age", int), ("score", float)]
DynClass = make_dataclass("DynClass", schema)

obj = DynClass(name="Alice", age=30, score=9.5)
print(obj)                    # DynClass(name='Alice', age=30, score=9.5)
print(is_dataclass(obj))      # True
print(DynClass(name="Alice", age=30, score=9.5) == obj)   # True — __eq__ works


# 6. make_dataclass with bases= and namespace=
def greet(self):
    return f"Hello, I am {self.name} ({self.role})"

ExtClass = make_dataclass(
    "ExtClass",
    [("role", str, field(default="user"))],
    bases=(DynClass,),
    namespace={"greet": greet},
)

e = ExtClass(name="Bob", age=25, score=8.0, role="admin")
print(e.greet())   # Hello, I am Bob (admin)
print(list(ExtClass.__dataclass_fields__))   # ['name', 'age', 'score', 'role']
```

---

## Exercise 6 — `replace()` for Copy-and-Update

```python
from dataclasses import dataclass, field, replace, fields
from datetime import datetime


@dataclass
class GameState:
    level:   int
    score:   int
    lives:   int
    players: list = field(default_factory=list)


# 1. Basic replace — original unchanged
gs = GameState(level=1, score=0, lives=3, players=["alice"])
gs2 = replace(gs, score=100)
print(gs.score)    # 0  — original
print(gs2.score)   # 100


# 2. Shallow copy problem — list is shared
gs3 = replace(gs, level=2)
gs3.players.append("bob")
print(gs.players)   # ['alice', 'bob'] — original ALSO mutated!


# 3. Fix: pass an explicit copy of the list
gs = GameState(level=1, score=0, lives=3, players=["alice"])
gs4 = replace(gs, level=2, players=list(gs.players))
gs4.players.append("bob")
print(gs.players)    # ['alice'] — clean


# 4. evolve() with field-name validation
MISSING = object()

def evolve(dc, **changes):
    valid = {f.name for f in fields(dc)}
    invalid = set(changes) - valid
    if invalid:
        raise TypeError(
            f"evolve() received unknown fields for "
            f"{type(dc).__name__}: {', '.join(sorted(invalid))}"
        )
    return replace(dc, **changes)

gs5 = evolve(gs, score=50)
print(gs5.score)   # 50

try:
    evolve(gs, typo=99)
except TypeError as e:
    print(e)


# 5. replace() re-runs __post_init__ — use init-visible field for set-once semantics
@dataclass
class Stamped:
    value:      int
    created_at: datetime = None   # init=True so replace() passes it through

    def __post_init__(self):
        if self.created_at is None:
            self.created_at = datetime.now()

s1 = Stamped(1)
s2 = replace(s1, value=2)
# replace() calls Stamped(value=2, created_at=s1.created_at)
# __post_init__ sees created_at is not None → skips the reset
print(s1.created_at == s2.created_at)   # True — preserved!


# 6. replace() on frozen — primary use case
@dataclass(frozen=True)
class Config:
    host: str = "localhost"
    port: int = 5432

cfg = Config()
cfg2 = replace(cfg, port=3306)
print(cfg2.port)   # 3306 — no FrozenInstanceError
print(cfg.port)    # 5432 — original unchanged
```

---

## Exercise 7 — `eq=False`, `order=True`, Custom Comparisons

```python
import heapq
from dataclasses import dataclass, field
from datetime import date


# 1 & 2. order=True — field-by-field, left to right (like tuple comparison)
@dataclass(order=True)
class Job:
    priority: int
    job_id:   str
    payload:  str = field(compare=False)   # excluded from == and ordering

jobs = [
    Job(2, "b-job", "payload"),
    Job(1, "c-job", "payload"),
    Job(1, "a-job", "payload"),
    Job(3, "d-job", "payload"),
]
sorted_jobs = sorted(jobs)
print([j.priority for j in sorted_jobs])    # [1, 1, 2, 3]
print([j.job_id   for j in sorted_jobs[:2]])  # ['a-job', 'c-job'] — tiebreak by job_id

heap = []
for j in jobs:
    heapq.heappush(heap, j)
print(heapq.heappop(heap).job_id)   # a-job


# 3. compare=False excludes from both == and ordering
j1 = Job(1, "same", "big_payload")
j2 = Job(1, "same", "small_payload")
print(j1 == j2)    # True — payload excluded


# 4. eq=False — no auto-generated __eq__; manual __eq__ makes class unhashable
# Fix: use unsafe_hash=True alongside eq=False when both are needed
@dataclass(eq=False, unsafe_hash=True)
class Node:
    value: int
    name:  str

    def __eq__(self, other):
        if not isinstance(other, Node): return NotImplemented
        return self.value == other.value   # compare only value

n1 = Node(1, "a")
n2 = Node(1, "b")
n3 = Node(2, "a")
print(n1 == n2)   # True  — custom: same value
print(n1 == n3)   # False — custom: different value
print(hash(n1) is not None)   # True — unsafe_hash preserves hashability

# Why manually defining __eq__ makes it unhashable by default:
# Python's rule: if you define __eq__, __hash__ is set to None unless
# you also explicitly define __hash__ or use unsafe_hash=True.


# 5. order=True, eq=False — ValueError at class definition
try:
    @dataclass(order=True, eq=False)
    class Invalid:
        x: int
except ValueError as e:
    print(e)   # eq must be true if order is true
# order=True generates __lt__ etc. in terms of __eq__; without __eq__ the
# ordering contract is broken.


# 6. Manual __lt__ with default eq=True
@dataclass
class Task:
    deadline: date
    title:    str

    def __lt__(self, other: "Task") -> bool:
        return self.deadline < other.deadline

tasks = [
    Task(date(2025, 3, 1), "Later task"),
    Task(date(2025, 1, 1), "Urgent task"),
]
print(sorted(tasks)[0].title)   # Urgent task
```

---

## Exercise 8 — Nested Dataclasses

```python
from dataclasses import dataclass, field, asdict, replace
from datetime import datetime
from decimal import Decimal


@dataclass
class Address:
    street:   str
    city:     str
    country:  str
    postcode: str

@dataclass
class Customer:
    customer_id: str
    name:        str
    email:       str
    billing:     Address
    shipping:    Address

@dataclass
class Product:
    sku:        str
    name:       str
    unit_price: Decimal

@dataclass
class LineItem:
    product:   Product
    quantity:  int
    discount:  float = 0.0

@dataclass
class Order:
    order_id:  str
    customer:  Customer
    items:     list
    placed_at: datetime = field(default_factory=datetime.now)

    @classmethod
    def from_dict(cls, data: dict) -> "Order":
        return cls(
            order_id=data["order_id"],
            customer=Customer(
                **{k: v for k, v in data["customer"].items()
                   if k not in ("billing", "shipping")},
                billing=Address(**data["customer"]["billing"]),
                shipping=Address(**data["customer"]["shipping"]),
            ),
            items=[
                LineItem(
                    product=Product(
                        unit_price=Decimal(str(i["product"]["unit_price"])),
                        **{k: v for k, v in i["product"].items() if k != "unit_price"},
                    ),
                    quantity=i["quantity"],
                    discount=i.get("discount", 0.0),
                )
                for i in data["items"]
            ],
            placed_at=datetime.fromisoformat(data["placed_at"])
            if isinstance(data.get("placed_at"), str)
            else data.get("placed_at", datetime.now()),
        )


addr   = Address("1 Main St", "London", "GB", "EC1A 1BB")
cust   = Customer("C1", "Alice", "alice@example.com", billing=addr, shipping=addr)
items  = [
    LineItem(Product("W001", "Widget", Decimal("9.99")), 3),
    LineItem(Product("G001", "Gadget", Decimal("29.99")), 1, discount=0.10),
]
order  = Order("ORD-001", cust, items)


# 1. Deep repr
print(repr(order)[:60], "...")


# 2. asdict — full recursive conversion
d = asdict(order)
print(type(d["items"][0]))            # dict
print(type(d["customer"]["billing"])) # dict


# 3. total_price
def total_price(order: Order) -> Decimal:
    return sum(
        item.product.unit_price
        * item.quantity
        * Decimal(str(1 - item.discount))
        for item in order.items
    )
print(total_price(order))   # 56.961 (9.99*3 + 29.99*0.9)


# 4. replace() on nested items
updated_item = replace(order.items[0], quantity=5)
new_order = replace(order, items=[updated_item] + order.items[1:])
print(new_order.items[0].quantity)   # 5
print(order.items[0].quantity)       # 3 — original unchanged

# Shallow copy bug: if items list is shared, appending to new_order.items
# would affect order.items — always copy the list explicitly:
safe_order = replace(order, items=list(order.items))


# 5. Partial freeze gotcha
@dataclass(frozen=True)
class FrozenItem:
    name: str
    qty:  int

@dataclass   # NOT frozen
class UnfrozenOrder:
    item: FrozenItem

uo = UnfrozenOrder(FrozenItem("Widget", 3))
uo.item = FrozenItem("Gadget", 5)   # allowed — UnfrozenOrder is mutable
print(uo.item.name)   # Gadget
# Fix: freeze EVERY layer, not just the inner ones


# 6. from_dict round-trip
d = asdict(order)
d["placed_at"] = order.placed_at.isoformat()   # convert for JSON
restored = Order.from_dict(d)
print(restored.order_id)             # ORD-001
print(restored.customer.name)        # Alice
print(len(restored.items))           # 2
```

---

## Exercise 9 — `InitVar`

```python
import os
from dataclasses import dataclass, field, asdict, astuple, InitVar


# Simulated DB connection
class MockConnection:
    def fetch_schema(self, table): return "public"


# 1. InitVar: used in __post_init__, absent from asdict
@dataclass
class DBRecord:
    table_name: str
    connection: InitVar["MockConnection"]
    schema:     str = field(init=False)
    full_path:  str = field(init=False)

    def __post_init__(self, connection: "MockConnection"):
        self.schema    = connection.fetch_schema(self.table_name)
        self.full_path = f"{self.schema}.{self.table_name}"

rec = DBRecord("users", MockConnection())
print(rec.full_path)                         # public.users
print(asdict(rec))                           # {'table_name': 'users', 'schema': 'public', 'full_path': 'public.users'}
# 'connection' absent from asdict output

# InitVar IS in __dataclass_fields__ but marked _FIELD_INITVAR
import dataclasses
print(DBRecord.__dataclass_fields__["connection"]._field_type)  # _FIELD_INITVAR


# 2. InitVar for raw config dict
@dataclass
class Config:
    raw_config: InitVar[dict]
    host:  str  = field(init=False)
    port:  int  = field(init=False)
    debug: bool = field(init=False)

    def __post_init__(self, raw_config: dict):
        self.host  = raw_config.get("host", "localhost")
        self.port  = int(raw_config.get("port", 5432))
        self.debug = bool(raw_config.get("debug", False))

cfg = Config(raw_config={"host": "db.prod", "port": "5433", "debug": "1"})
print(cfg.host, cfg.port, cfg.debug)   # db.prod 5433 True


# 3. Optional InitVar with default None
@dataclass
class Enrichable:
    value:    str
    enricher: InitVar[dict | None] = None
    extra:    str = field(init=False, default="")

    def __post_init__(self, enricher):
        if enricher is not None:
            self.extra = enricher.get("label", "")

e1 = Enrichable("hello")
e2 = Enrichable("hello", enricher={"label": "greeting"})
print(e1.extra)   # ''
print(e2.extra)   # 'greeting'


# 4. Invisible to asdict, repr, astuple
print("raw_config" in asdict(cfg))   # False
print("raw_config" in repr(cfg))     # False
print("raw_config" in str(astuple(cfg)))  # False (value not there)


# 5. InitVar in inheritance — both InitVars appear in Child.__init__
@dataclass
class BaseInit:
    name:       str
    multiplier: InitVar[int] = 1
    scaled:     float = field(init=False)

    def __post_init__(self, multiplier: int):
        self.scaled = len(self.name) * multiplier


@dataclass
class ChildInit(BaseInit):
    suffix:     str       = ""
    extra_mult: InitVar[int] = 1
    full_name:  str = field(init=False)

    def __post_init__(self, multiplier: int, extra_mult: int):
        super().__post_init__(multiplier)   # pass base's InitVar up
        self.full_name = self.name + self.suffix * extra_mult

ci = ChildInit("hello", multiplier=2, suffix="!", extra_mult=3)
print(ci.scaled)      # 10 (5 chars × 2)
print(ci.full_name)   # hello!!!


# 6. from_env() alternative — compare with InitVar
@dataclass
class ServerConfig:
    host:  str
    port:  int
    debug: bool

    @classmethod
    def from_env(cls) -> "ServerConfig":
        return cls(
            host  = os.environ.get("HOST", "localhost"),
            port  = int(os.environ.get("PORT", "8080")),
            debug = os.environ.get("DEBUG", "").lower() in ("1", "true"),
        )

# Tradeoffs:
# InitVar:      construction is always via raw data; no separate factory needed;
#               harder to discover; doesn't appear in signature tools clearly.
# from_env():   explicit factory; clear call site; supports multiple construction
#               paths (from_env, from_dict, direct); preferred for optional sources.
sc = ServerConfig.from_env()
print(sc)
```

---

## Exercise 10 — `field(init=False)` for Computed Attributes

```python
from dataclasses import dataclass, field, asdict
from datetime import datetime


# 1. Computed metrics fields
@dataclass
class SalesMetrics:
    revenue:         float
    cost:            float
    units_sold:      int
    gross_profit:    float = field(init=False)
    margin_pct:      float = field(init=False)
    profit_per_unit: float = field(init=False)

    def __post_init__(self):
        self.gross_profit    = self.revenue - self.cost
        self.margin_pct      = (self.gross_profit / self.revenue * 100) if self.revenue else 0.0
        self.profit_per_unit = (self.gross_profit / self.units_sold) if self.units_sold else 0.0

sm = SalesMetrics(10_000.0, 6_000.0, 200)
print(sm.gross_profit)     # 4000.0
print(sm.margin_pct)       # 40.0
print(sm.profit_per_unit)  # 20.0


# 2. init=False fields appear in repr, ==, asdict
print("gross_profit" in repr(sm))       # True
print("gross_profit" in asdict(sm))     # True
print(sm == SalesMetrics(10_000.0, 6_000.0, 200))  # True


# 3. field(init=False) requires a default — show the error
try:
    @dataclass
    class Broken:
        x: float
        derived: float = field(init=False)   # no default → TypeError
except TypeError as e:
    print(e)

@dataclass
class Fixed:
    x: float
    derived: float = field(init=False, default=0.0)   # has default
    def __post_init__(self): self.derived = self.x * 2


# 4. Timestamped base — field ordering constraint
@dataclass
class ProductFixed:
    sku:        str
    name:       str
    # Timestamps go AFTER all non-default fields because they have defaults
    created_at: datetime = field(init=False, default_factory=datetime.now)
    updated_at: datetime = field(init=False, default_factory=datetime.now)

pf = ProductFixed("P001", "Widget")
print(pf.created_at.date())   # today


# 5. frozen=True + field(init=False) → object.__setattr__ required
@dataclass(frozen=True)
class FrozenMetrics:
    revenue: float
    cost:    float
    profit:  float = field(init=False)

    def __post_init__(self):
        # self.profit = ...  → FrozenInstanceError
        object.__setattr__(self, "profit", self.revenue - self.cost)

fm = FrozenMetrics(1000.0, 600.0)
print(fm.profit)   # 400.0


# 6. Private cache field — invisible to repr/eq, but present in asdict
@dataclass
class ExpensiveModel:
    data: list
    _cache: dict = field(
        init=False,
        repr=False,
        compare=False,
        default_factory=dict,
    )

    def compute(self, key: str) -> int:
        if key not in self._cache:
            self._cache[key] = sum(x for x in self.data if str(x) == key)
        return self._cache[key]

em = ExpensiveModel([1, 2, 1, 3, 1])
print(em.compute("1"))                    # 3
print(em.compute("1"))                    # 3 — from cache
print("_cache" in repr(em))              # False
print(em == ExpensiveModel([1, 2, 1, 3, 1]))  # True — _cache excluded

# Note: _cache IS included in asdict() because compare=False ≠ exclude from asdict
# To fully exclude from asdict, filter after the call:
d = {k: v for k, v in asdict(em).items() if not k.startswith("_")}
print("_cache" in d)   # False
```

---

## Bonus — Bond Trading Platform

```python
from dataclasses import dataclass, field, replace, asdict
from datetime import date, datetime
from decimal import Decimal, ROUND_HALF_UP


@dataclass(frozen=True)
class Money:
    amount:   Decimal
    currency: str

    def __post_init__(self):
        if not (len(self.currency) == 3 and self.currency.isalpha()):
            raise ValueError(f"currency must be a 3-letter ISO code: {self.currency!r}")
        object.__setattr__(
            self, "amount",
            self.amount.quantize(Decimal("0.01"), ROUND_HALF_UP)
        )

    def __add__(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError(f"Cannot add {self.currency} and {other.currency}")
        return Money(self.amount + other.amount, self.currency)

    def __mul__(self, scalar) -> "Money":
        return Money(self.amount * Decimal(str(scalar)), self.currency)

    def __lt__(self, other: "Money") -> bool:
        return self.amount < other.amount

    def to_dict(self) -> dict:
        return {"amount": str(self.amount), "currency": self.currency}

    @classmethod
    def from_dict(cls, data: dict) -> "Money":
        return cls(Decimal(data["amount"]), data["currency"])


@dataclass(frozen=True)
class BondSpec:
    isin:         str
    coupon_rate:  float
    maturity:     date
    notional:     Money
    country_code: str = field(init=False)
    issuer_code:  str = field(init=False)

    def __post_init__(self):
        if len(self.isin) != 12:
            raise ValueError(f"ISIN must be 12 characters: {self.isin!r}")
        object.__setattr__(self, "country_code", self.isin[:2])
        object.__setattr__(self, "issuer_code",  self.isin[2:11])


# Auditable: all fields are init-visible (default=None/0) so replace() carries them
@dataclass
class Auditable:
    created_at: datetime = None
    updated_at: datetime = None
    version:    int      = 0


@dataclass
class TradeTicket(Auditable):
    bond:     BondSpec = None
    side:     str      = "BUY"
    quantity: int      = 0

    def __post_init__(self):
        # Set timestamps only on first construction
        if self.created_at is None:
            self.created_at = datetime.now()
        if self.updated_at is None:
            self.updated_at = datetime.now()
        # version is carried through by replace(), then incremented here
        self.version += 1


def amend(ticket: TradeTicket, **changes) -> TradeTicket:
    """Return a new TradeTicket with changes applied and version incremented."""
    amended = replace(ticket, **changes)
    amended.updated_at = datetime.now()
    return amended


# ── Demo ─────────────────────────────────────────────────────
spec = BondSpec(
    isin="US912810TD00",
    coupon_rate=0.0425,
    maturity=date(2053, 2, 15),
    notional=Money(Decimal("1000000"), "USD"),
)
print(f"country: {spec.country_code}, issuer: {spec.issuer_code}")

ticket = TradeTicket(bond=spec, side="BUY", quantity=10)
print(f"version: {ticket.version}")      # 1

amended = amend(ticket, quantity=20)
print(f"amended qty:     {amended.quantity}")   # 20
print(f"amended version: {amended.version}")    # 2
print(f"original qty:    {ticket.quantity}")    # 10 — unchanged
print(f"different objs:  {amended is not ticket}")  # True

# Money arithmetic
total = spec.notional * amended.quantity
print(f"total exposure: {total.amount} {total.currency}")

# Round-trip serialisation
bond_dict = {
    "isin":        spec.isin,
    "coupon_rate": spec.coupon_rate,
    "maturity":    spec.maturity.isoformat(),
    "notional":    spec.notional.to_dict(),
}
restored = BondSpec(
    isin=bond_dict["isin"],
    coupon_rate=bond_dict["coupon_rate"],
    maturity=date.fromisoformat(bond_dict["maturity"]),
    notional=Money.from_dict(bond_dict["notional"]),
)
print(f"round-trip match: {restored.isin == spec.isin}")   # True
```
