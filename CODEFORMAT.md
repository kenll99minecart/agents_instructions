---
name: it-cisq-maintainability
description: Review, refactor, and write source code against the 29 maintainability rules of the CISQ/IT-CISQ coding standards (CWE-407, 478, 480, 484, 561, 570, 571, 783, 1041, 1045, 1047, 1048, 1051, 1052, 1054, 1055, 1062, 1064, 1074, 1075, 1079, 1080, 1084, 1085, 1086, 1087, 1090, 1095, 1121). Use this whenever you are auditing code quality, doing a code review, reducing complexity, breaking up god classes or god files, untangling inheritance, removing dead or duplicated code, refactoring long parameter lists, or when the user mentions CISQ, CWE maintainability, structural quality, technical debt, cyclomatic complexity, coupling, cohesion, or "why is this code hard to change". Also use proactively before declaring any non-trivial refactor complete.
---

# CISQ Maintainability Rules

The CISQ structural-quality standards score software across four characteristics: Reliability, Performance Efficiency, Security, and **Maintainability**. Maintainability is the one that decides whether a codebase can still be changed cheaply two years from now — the other three describe what the running program does, this one describes what it costs a human to modify it.

This skill covers all **29 maintainability rules**. Each is a CWE weakness that CISQ-certified tools detect in source code.

Source: https://www.it-cisq.org/coding-rules/ — the page publishes rule identifiers and names only, with no examples. Every code sample below is written against the CWE definition, not copied from CISQ.

## How to use this skill

**Reviewing code.** Walk the target through the checklist in the order below — grouped by theme so related violations surface together. Report findings as `CWE-<id> at <file>:<line>` with the specific fix, not a generic "this is complex". Prefer naming the smallest change that removes the violation.

**Writing code.** Skim the checklist before you start a module. Most of these rules are cheap to satisfy during design and expensive to retrofit.

**A note on thresholds.** CISQ does not publish numeric cutoffs for "excessive" — those come from your tool configuration and your team's norms. Where a rule is about size or count, the examples below use commonly configured limits (cyclomatic complexity ≤ 10, parameters ≤ 5, file ≤ 1000 LOC, inheritance depth ≤ 5). Treat them as defaults to argue about, not laws.

## Quick reference

| Theme | Rules |
|---|---|
| Control flow & logic errors | 478, 480, 484, 570, 571, 783, 1075, 1095 |
| Dead & duplicated code | 561, 1041, 1085 |
| Complexity & size | 407, 1121, 1064, 1080, 1048, 1084 |
| Coupling & layering | 1047, 1054, 1062, 1090 |
| Inheritance design | 1045, 1055, 1074, 1079, 1086, 1087 |
| Hard-coded values | 1051, 1052 |

---

## Control flow & logic errors

These read as "the code doesn't do what the author thought". They survive review because each individual line looks fine.

### CWE-478 — Missing Default Case in Switch Statement

A `switch` with no `default` silently does nothing when a value outside the enumerated cases arrives. Enum members get added, the compiler stays quiet, and the new case becomes a no-op that surfaces as a mysterious bug far from this file. The `default` is where you make the unhandled case loud.

```java
// BAD — a fourth ShipmentMethod added later falls through silently
switch (method) {
    case GROUND:  cost = weight * 0.5; break;
    case AIR:     cost = weight * 2.0; break;
    case FREIGHT: cost = weight * 0.1; break;
}

// GOOD — unknown values fail immediately, at the point of the mistake
switch (method) {
    case GROUND:  cost = weight * 0.5; break;
    case AIR:     cost = weight * 2.0; break;
    case FREIGHT: cost = weight * 0.1; break;
    default:
        throw new IllegalStateException("Unhandled ShipmentMethod: " + method);
}
```

### CWE-480 — Use of Incorrect Operator

The right shape of statement, the wrong symbol. Assignment where comparison was meant, bitwise where logical was meant, string concatenation where multiplication was meant. These pass compilation and often pass a happy-path test.

```c
// BAD — assignment, not comparison; body always runs, and count is destroyed
if (count = 0) {
    reset_buffer();
}

// GOOD
if (count == 0) {
    reset_buffer();
}
```

```python
# BAD — `is` compares identity; works for small ints by accident, fails elsewhere
if user.role is "admin":
    grant_access()

# GOOD
if user.role == "admin":
    grant_access()
```

### CWE-484 — Omitted Break Statement in Switch

Fallthrough. In C-family languages it is the default behaviour, so omitting `break` is a silent transfer of control into the next case. When fallthrough is genuinely intended, say so in a comment or use the explicit fallthrough attribute — an unannotated fallthrough is indistinguishable from a typo.

```c
// BAD — a paid order also gets marked cancelled
switch (status) {
    case PENDING:
        notify_customer();
    case PAID:
        schedule_shipment();
    case CANCELLED:
        release_inventory();
}

// GOOD
switch (status) {
    case PENDING:
        notify_customer();
        break;
    case PAID:
        schedule_shipment();
        break;
    case CANCELLED:
        release_inventory();
        break;
}
```

### CWE-570 — Expression is Always False

The condition can never be true, so the guarded branch is unreachable. Usually a type mismatch — an unsigned value tested for negativity, a range check that already excludes the case. The author believed there was a path here; there isn't, and the real bug is upstream.

```c
// BAD — size_t is unsigned; a failed read yields a huge value, not -1
size_t n = fread(buf, 1, sizeof(buf), f);
if (n < 0) {          // always false
    handle_error();
}

// GOOD
if (n == 0 || ferror(f)) {
    handle_error();
}
```

### CWE-571 — Expression is Always True

The mirror of CWE-570. A guard that never fires gives false confidence: the error handling looks present but can never run, so real failures pass through unchecked.

```c
// BAD — `new` throws on failure; it never returns null, so the check is inert
Widget* w = new Widget();
if (w != nullptr) {   // always true
    w->render();
}

// GOOD — check the condition that can actually fail
Widget* w = new (std::nothrow) Widget();
if (w == nullptr) {
    log_allocation_failure();
    return;
}
w->render();
```

### CWE-783 — Operator Precedence Logic Error

The expression is parsed differently from how it reads. In C, `==` binds tighter than `&`, so `flags & MASK == 0` tests `flags & (MASK == 0)`. Adding parentheses costs nothing and removes the need for the reader to recall a precedence table.

```c
// BAD — evaluates as flags & (0x0F == 0), i.e. flags & 0
if (flags & 0x0F == 0) {
    report_clean();
}

// GOOD
if ((flags & 0x0F) == 0) {
    report_clean();
}
```

```python
# BAD — `not` binds tighter than `in`: parses as (not x) in collection
if not x in allowed:
    reject()

# GOOD
if x not in allowed:
    reject()
```

### CWE-1075 — Unconditional Control Flow Transfer outside of Switch Block

A `return`, `break`, `goto`, or `throw` that executes regardless of any condition, positioned so that code after it can never run. Outside a `switch` there is no legitimate fallthrough semantics to lean on — the statement is either misplaced or the code after it is dead.

```python
# BAD — the retry branch is unreachable
def fetch(url):
    try:
        return http_get(url)
    except Timeout:
        return None
    return http_get(url, retry=True)   # never executes

# GOOD
def fetch(url):
    try:
        return http_get(url)
    except Timeout:
        return http_get(url, retry=True)
```

### CWE-1095 — Loop Condition Value Update within the Loop

The loop variable is also modified inside the body, so the number of iterations no longer follows from the header. Readers reason about loops from the `for` line; breaking that contract turns a three-line loop into something that needs tracing. Use a `while` with an explicit step if the stride really is dynamic.

```python
# BAD — i is advanced in two places; which one wins depends on the data
for i in range(len(rows)):
    if rows[i].is_merged:
        i += 1          # skipped cell, but range() ignores it
    total += rows[i].value

# GOOD — make the skip explicit and keep one owner of the index
i = 0
while i < len(rows):
    if rows[i].is_merged:
        i += 1
        continue
    total += rows[i].value
    i += 1
```

---

## Dead & duplicated code

Dead code is not harmless clutter. It is read, reasoned about, maintained, and refactored by everyone who comes after — and it misleads them about what the system actually does.

### CWE-561 — Dead Code

Code that is present but can never affect the result: unreachable branches, unused variables, functions nothing calls. Version control remembers it; the source tree should not.

```python
# BAD — validate() is never called, and `normalized` is written but never read
def process(raw):
    normalized = raw.strip().lower()
    return parse(raw)

def validate(text):        # no callers anywhere in the codebase
    return len(text) > 0

# GOOD
def process(raw):
    return parse(raw.strip().lower())
```

### CWE-1041 — Use of Redundant Code (Copy-Paste)

The same logic in two or more places. The cost is not the duplication itself but that every future change must be found and applied N times, and the copies always drift. Extract once, name it after the concept it implements.

```javascript
// BAD — the same rule in three handlers; they will diverge
function onSignup(form)  { if (!/^\S+@\S+\.\S+$/.test(form.email)) throw new Error("bad email"); }
function onUpdate(form)  { if (!/^\S+@\S+\.\S+$/.test(form.email)) throw new Error("bad email"); }
function onInvite(form)  { if (!/^\S+@\S+\.\S+$/.test(form.email)) throw new Error("bad email"); }

// GOOD
const EMAIL = /^\S+@\S+\.\S+$/;
function assertValidEmail(email) {
    if (!EMAIL.test(email)) throw new Error(`Invalid email: ${email}`);
}

function onSignup(form) { assertValidEmail(form.email); }
function onUpdate(form) { assertValidEmail(form.email); }
function onInvite(form) { assertValidEmail(form.email); }
```

### CWE-1085 — Invokable Control Element with Excessive Volume of Commented-out Code

Large commented-out blocks inside a function. They look like documentation but read as "an alternative implementation still under consideration", so the next person cannot tell whether deleting them loses something. Commit history is the archive; use it.

```python
# BAD
def score(review):
    base = review.rating * 10
    # base = review.rating * 12
    # if review.verified:
    #     base += 5
    # base = min(base, 100)
    # ---- old v1 scoring ----
    # return legacy_score(review.text)
    return base + review.bonus

# GOOD — a one-line note explains the decision; history holds the code
def score(review):
    # v2 scoring: verification bonus moved to review.bonus (see #4821)
    return review.rating * 10 + review.bonus
```

---

## Complexity & size

These are the rules that quantify "hard to change". They are the ones a CISQ score weights most heavily, and the ones where the fix is almost always decomposition rather than cleverness.

### CWE-407 — Algorithmic Complexity

An algorithm whose cost grows faster than the problem requires. Distinct from cyclomatic complexity: here the code may be short and clear, and still be quadratic. Nested scans over the same collection are the most common source.

```python
# BAD — O(n^2): rescans the whole list for every element
def find_duplicates(items):
    dupes = []
    for i, a in enumerate(items):
        for b in items[i + 1:]:
            if a == b and a not in dupes:
                dupes.append(a)
    return dupes

# GOOD — O(n)
from collections import Counter

def find_duplicates(items):
    return [item for item, n in Counter(items).items() if n > 1]
```

### CWE-1121 — Excessive McCabe Cyclomatic Complexity

Too many independent paths through one function. Each branch is a path that needs its own test and its own mental slot, so complexity is roughly the minimum test count for full branch coverage. Above ~10, extract the branches into named predicates or push them into a dispatch table.

```python
# BAD — one function, ~12 paths
def price(order):
    if order.region == "EU":
        if order.total > 100:
            if order.member:
                return order.total * 0.8
            else:
                return order.total * 0.9
        elif order.member:
            return order.total * 0.95
        else:
            return order.total
    elif order.region == "US":
        if order.total > 100:
            return order.total * 0.85
        else:
            return order.total
    else:
        return order.total * 1.1

# GOOD — each decision named and independently testable
REGION_UPLIFT = {"EU": 1.0, "US": 1.0}
DEFAULT_UPLIFT = 1.1

def price(order):
    discount = _discount_factor(order)
    uplift = REGION_UPLIFT.get(order.region, DEFAULT_UPLIFT)
    return order.total * discount * uplift

def _discount_factor(order):
    if order.region not in REGION_UPLIFT:
        return 1.0
    if order.total > 100:
        return 0.8 if order.member else (0.85 if order.region == "US" else 0.9)
    return 0.95 if order.member else 1.0
```

### CWE-1064 — Invokable Control Element with Signature Containing an Excessive Number of Parameters

A long parameter list. Call sites become positional puzzles, optional combinations are unclear, and every new option changes the signature and all callers. Group parameters that change together into a value object — the grouping usually names a concept that was missing from the design.

```python
# BAD — 9 positional args; easy to transpose two strings and get no error
def create_user(first, last, email, phone, street, city, state, zip_code, country):
    ...

create_user("Ada", "Lovelace", "ada@x.io", "555-0100",
            "12 Main St", "London", "LDN", "EC1", "UK")

# GOOD
@dataclass(frozen=True)
class Address:
    street: str
    city: str
    region: str
    postal_code: str
    country: str

@dataclass(frozen=True)
class Contact:
    email: str
    phone: str

def create_user(name: FullName, contact: Contact, address: Address) -> User:
    ...

create_user(
    FullName("Ada", "Lovelace"),
    Contact(email="ada@x.io", phone="555-0100"),
    Address("12 Main St", "London", "LDN", "EC1", "UK"),
)
```

### CWE-1080 — Source Code File with Excessive Number of Lines of Code

A single file carrying too much. File size is a proxy: it usually means several responsibilities were merged, and it makes navigation, review diffs, and merge conflicts all worse. Split along the seams that already exist — the comment banners and section groupings are the author telling you where the boundaries are.

```
# BAD
# src/utils.py — 3,400 lines
#   - date helpers
#   - HTTP retry wrapper
#   - CSV parsing
#   - logging setup
#   - the entire pricing engine
from utils import parse_csv, retry_http, calculate_price   # nobody knows what's in here

# GOOD
src/
  dates.py
  http.py
  csvio.py
  logging_setup.py
  pricing/
    engine.py
    rules.py
```

### CWE-1048 — Invokable Control Element with Large Number of Outward Calls (Excessive Coupling / Fan-out)

One function that reaches into many other components. High fan-out means the function cannot be understood, tested, or changed without all of its collaborators present — a change in any one of them can break it. Extract the coordination into a service and let the function depend on one thing.

```python
# BAD — generate_report touches 7 unrelated subsystems
def generate_report(order_id):
    order = order_repo.find(order_id)
    user = user_service.get(order.user_id)
    lines = inventory_api.fetch_lines(order.skus)
    tax = tax_client.compute(order.region, lines)
    pdf = renderer.render(order, user, lines, tax)
    blob_store.put(f"reports/{order_id}.pdf", pdf)
    mailer.send(user.email, pdf)
    metrics.increment("reports.generated")

# GOOD — one collaborator per seam; the orchestrator only orchestrates
class ReportService:
    def __init__(self, compiler, storage, notifier, metrics):
        self.compiler = compiler
        self.storage = storage
        self.notifier = notifier
        self.metrics = metrics

    def generate(self, order_id):
        report = self.compiler.compile(order_id)
        self.storage.save(report)
        self.notifier.deliver(report)
        self.metrics.increment("reports.generated")
        return report
```

### CWE-1084 — Invokable Control Element with Excessive File or Data Access Operations

Too many I/O calls inside one function — the data-access cousin of excessive fan-out. Beyond coupling, this is where N+1 query patterns are born: a loop issuing one query per row instead of one query for the set.

```python
# BAD — 1 + N queries, and I/O is tangled with business logic
def order_summary(ids):
    total = 0
    for oid in ids:
        order = db.query("SELECT * FROM orders WHERE id = %s", oid)
        for line in db.query("SELECT * FROM lines WHERE order_id = %s", oid):
            total += line.qty * line.price
    return total

# GOOD — one round trip; aggregation belongs to the data layer
def order_summary(ids):
    return db.query_one(
        """SELECT COALESCE(SUM(l.qty * l.price), 0)
             FROM orders o JOIN lines l ON l.order_id = o.id
            WHERE o.id = ANY(%s)""",
        ids,
    )
```

---

## Coupling & layering

These describe dependencies that exist but should not. They are invisible in any single file — you only see them by looking at what imports what.

### CWE-1047 — Modules with Circular Dependencies

A depends on B and B depends on A. Neither can be built, tested, reused, or removed alone, and the pair behaves as one module with two names. Break the cycle by extracting the shared concept into a third module both may depend on, or by inverting one direction through an interface.

```python
# BAD
# orders.py
from billing import charge_card        # orders -> billing
def place_order(cart):
    charge_card(cart.total)

# billing.py
from orders import OrderLog            # billing -> orders  (cycle)
def charge_card(amount):
    OrderLog.record(amount)

# GOOD — the shared concept moves to a third module both depend on
# ledger.py            (depends on nothing)
class OrderLog: ...

# orders.py            -> billing, ledger
from billing import charge_card
def place_order(cart):
    charge_card(cart.total)

# billing.py           -> ledger
from ledger import OrderLog
def charge_card(amount):
    OrderLog.record(amount)
```

### CWE-1054 — Invocation of a Control Element at an Unnecessarily Deep Horizontal Layer (Layer-skipping Call)

A caller bypasses the layer directly above it — a controller reaching past the service into the repository. The skipped layer is where validation, authorization, and transaction boundaries normally live, so the bypass quietly loses them. Every call should go one layer down, never two.

```java
// BAD — controller reaches past the service layer into the repository,
// skipping the authorization and validation that OrderService applies
@RestController
class OrderController {
    @Autowired OrderRepository repo;

    @PostMapping("/orders/{id}/cancel")
    void cancel(@PathVariable String id) {
        repo.updateStatus(id, "CANCELLED");   // no auth check, no inventory release
    }
}

// GOOD
@RestController
class OrderController {
    private final OrderService orders;

    @PostMapping("/orders/{id}/cancel")
    void cancel(@PathVariable String id, @Auth User user) {
        orders.cancel(id, user);
    }
}
```

### CWE-1062 — Parent Class Element with References to Child Class

A base class that knows about its subclasses. This inverts the dependency that inheritance is supposed to create: the abstraction now depends on its implementations, so adding a subclass requires editing the parent, and the parent can never be reused without dragging the children along. Dispatch belongs to the subclasses or to a registry outside the hierarchy.

```python
# BAD — Shape must be edited every time a new subclass is added
class Shape:
    def area(self):
        if isinstance(self, Square):
            return self.side ** 2
        elif isinstance(self, Circle):
            return math.pi * self.radius ** 2
        raise TypeError("unknown shape")

# GOOD — each subclass owns its own rule; Shape never mentions them
class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

class Square(Shape):
    def area(self):
        return self.side ** 2

class Circle(Shape):
    def area(self):
        return math.pi * self.radius ** 2
```

### CWE-1090 — Method Containing Access of a Member Element from Another Class

Reaching directly into another object's fields rather than asking it through a method. The owner class loses control of its own invariants — it cannot validate, lazily compute, or change the representation without breaking every accessor. This is the Law of Demeter expressed as a rule: talk to your immediate collaborators, not their internals.

```python
# BAD — three levels deep, and Cart cannot change how it stores lines
def total_with_tax(cart):
    subtotal = sum(line.price * line.qty for line in cart.items.lines)
    return subtotal * (1 + cart.items.lines[0].seller.region.tax_rate)

# GOOD — each object answers for itself
def total_with_tax(cart):
    return cart.subtotal() * (1 + cart.tax_rate())
```

---

## Inheritance design

Inheritance is the tightest coupling available in an OO language: the child sees the parent's implementation, not just its interface. These six rules are about paying that cost only where it is earned.

### CWE-1079 — Parent Class without Virtual Destructor Method

A polymorphic base class whose destructor is not virtual. Deleting a derived object through a base pointer then runs only the base destructor — the derived members are never released. This is undefined behaviour that typically shows up as a slow leak in production, not a crash in test.

```cpp
// BAD — deleting through Base* skips ~Derived(); the buffer leaks
class Base {
public:
    ~Base() {}
};
class Derived : public Base {
    char* buf = new char[4096];
public:
    ~Derived() { delete[] buf; }
};

Base* p = new Derived();
delete p;                       // undefined behaviour

// GOOD
class Base {
public:
    virtual ~Base() = default;
};
```

If a class is never deleted polymorphically, the cheaper alternative is a non-virtual **protected** destructor, which makes `delete basePtr` a compile error instead of a runtime leak.

### CWE-1087 — Class with Virtual Method without a Virtual Destructor

The same defect detected from the other direction: a class declares at least one virtual method — so it is intended to be used polymorphically — but its destructor is not virtual. Any class with virtual methods is a polymorphic base by design, and needs the matching destructor.

```cpp
// BAD — virtual methods announce polymorphic use; the destructor contradicts it
class Renderer {
public:
    virtual void draw() = 0;
    ~Renderer() { release_context(); }
};

std::unique_ptr<Renderer> r = std::make_shared<GLRenderer>();
r.reset();                      // ~GLRenderer never runs

// GOOD
class Renderer {
public:
    virtual void draw() = 0;
    virtual ~Renderer() { release_context(); }
};
```

### CWE-1045 — Parent Class with a Virtual Destructor and a Child Class without a Virtual Destructor

The parent establishes the contract that this hierarchy is destroyed polymorphically; a child that declares its own destructor without `virtual` breaks the pattern's consistency. In C++ the child's destructor is implicitly virtual, so this rarely breaks behaviour — it breaks readability, and it hides the intent from the next person adding a grandchild class in a language where the rule differs.

```cpp
// BAD — parent declares the polymorphic contract, child does not honour it visibly
class Connection {
public:
    virtual ~Connection() { close(); }
};
class PooledConnection : public Connection {
public:
    ~PooledConnection() { release_to_pool(); }   // virtual intent not stated
};

// GOOD
class PooledConnection : public Connection {
public:
    virtual ~PooledConnection() override { release_to_pool(); }
};
```

### CWE-1055 — Multiple Inheritance from Concrete Classes

Inheriting implementation from more than one concrete class. Two parents bring two sets of fields, two initialisation orders, and — when they share an ancestor — the diamond problem, where the same base subobject exists twice. Inherit implementation from at most one class; take additional capabilities through interfaces or composition.

```cpp
// BAD — Logger and Timer both derive from Component; Widget holds two copies
class Component { public: int id; };
class Logger : public Component { /* ... */ };
class Timer  : public Component { /* ... */ };

class Widget : public Logger, public Timer {
    // Widget::id is ambiguous — Logger::id or Timer::id?
};

// GOOD — one implementation parent, capabilities by composition
class Widget : public Component {
    Logger log_;
    Timer timer_;
};
```

### CWE-1074 — Class with Excessively Deep Inheritance

An inheritance chain longer than about five levels. Behaviour arrives from so many places that reading the leaf class tells you almost nothing — you must walk the whole chain, and each level can override what the one above did. Flatten it: pull shared state into a composed component, or replace the chain with a strategy.

```python
# BAD — 6 levels; where does save() actually come from?
class Entity: ...
class PersistedEntity(Entity): ...
class TimestampedEntity(PersistedEntity): ...
class VersionedEntity(TimestampedEntity): ...
class AuditedEntity(VersionedEntity): ...
class SoftDeleteEntity(AuditedEntity): ...
class Customer(SoftDeleteEntity): ...

# GOOD — one level of inheritance, the rest composed and explicit
class Customer(Entity):
    def __init__(self, ...):
        self.persistence = PersistencePolicy(table="customers")
        self.audit = AuditTrail()
        self.versioning = Versioning()

    def save(self):
        self.versioning.bump()
        self.persistence.write(self)
        self.audit.record("save", self)
```

### CWE-1086 — Class with Excessive Number of Child Classes

A base class with very many subclasses. High subclass fan-out means the parent's contract is load-bearing for a large surface: any change to it risks breaking all of them, and the parent tends to accrete special-case hooks to satisfy the outliers. When the children differ mostly in *data* rather than *behaviour*, they were probably never classes — collapse them into one class parameterised by configuration.

```python
# BAD — 20 subclasses that differ only in two constants
class PdfExporter(Exporter):    format = "pdf";  mime = "application/pdf"
class HtmlExporter(Exporter):   format = "html"; mime = "text/html"
class CsvExporter(Exporter):    format = "csv";  mime = "text/csv"
# ... 17 more

# GOOD — one class, the variation becomes data
EXPORTERS = {
    "pdf":  ExportSpec(mime="application/pdf",  writer=write_pdf),
    "html": ExportSpec(mime="text/html",        writer=write_html),
    "csv":  ExportSpec(mime="text/csv",         writer=write_csv),
}

class Exporter:
    def __init__(self, fmt: str):
        self.spec = EXPORTERS[fmt]

    def export(self, rows):
        return self.spec.writer(rows)
```

---

## Hard-coded values

### CWE-1051 — Initialization with Hard-Coded Network Resource Configuration Data

An IP address, hostname, port, or URL baked into source. The value differs between environments by definition, so hard-coding it forces either a source edit per deploy or a branch per environment — and it means the same binary cannot be promoted from staging to production. Configuration belongs outside the code, with the code holding only the default.

```python
# BAD — promoting this to production requires editing source
DB_HOST = "10.0.3.47"
DB_PORT = 5432
conn = connect("postgres://svc:pw@10.0.3.47:5432/orders")

# GOOD — resolved from the environment at startup, defaults documented
DB_HOST = os.environ.get("DB_HOST", "localhost")
DB_PORT = int(os.environ.get("DB_PORT", "5432"))
conn = connect(os.environ["DATABASE_URL"])
```

### CWE-1052 — Excessive Use of Hard-Coded Literals in Initialization

Magic numbers and strings scattered through initialisation code. The problem is not that a literal exists but that its meaning is unstated and its value is duplicated: a business constant that appears in six files will be updated in five. Give it a name that carries the reason, in one place.

```python
# BAD — what is 86400? and is the 300 on line 40 the same 300 as here?
cache.set(key, value, 300)
if time.time() - last > 86400:
    notify()
retries = 3

# GOOD — the name states the intent, the value has one home
CACHE_TTL_SECONDS = 300
NOTIFICATION_COOLDOWN_SECONDS = 24 * 60 * 60   # one notification per day
MAX_UPLOAD_RETRIES = 3

cache.set(key, value, CACHE_TTL_SECONDS)
if time.time() - last > NOTIFICATION_COOLDOWN_SECONDS:
    notify()
retries = MAX_UPLOAD_RETRIES
```

---

## Rule index

| CWE | Name | One-line fix |
|---|---|---|
| 407 | Algorithmic Complexity | Replace the quadratic scan with a hash lookup or better algorithm |
| 478 | Missing Default Case in Switch Statement | Add a `default` that throws or logs |
| 480 | Use of Incorrect Operator | `=` → `==`, `&` → `&&`, `is` → `==` |
| 484 | Omitted Break Statement in Switch | Add `break`; annotate intentional fallthrough |
| 561 | Dead Code | Delete it; version control is the archive |
| 570 | Expression is Always False | Fix the type or the range being tested |
| 571 | Expression is Always True | Test the condition that can actually fail |
| 783 | Operator Precedence Logic Error | Parenthesise every mixed-precedence expression |
| 1041 | Use of Redundant Code (Copy-Paste) | Extract the concept into one named function |
| 1045 | Parent with Virtual Destructor, Child without | Declare the child destructor `virtual ... override` |
| 1047 | Modules with Circular Dependencies | Extract the shared concept into a third module |
| 1048 | Excessive Coupling / Fan-out | Inject collaborators; let the function orchestrate only |
| 1051 | Hard-Coded Network Resource Config | Read host/port/URL from environment or config |
| 1052 | Excessive Hard-Coded Literals | Name the constant once, where its reason is visible |
| 1054 | Layer-skipping Call | Call the adjacent layer, never two down |
| 1055 | Multiple Inheritance from Concrete Classes | One implementation parent; compose the rest |
| 1062 | Parent Class with References to Child Class | Push dispatch into the subclasses |
| 1064 | Excessive Number of Parameters | Group parameters that change together into a value object |
| 1074 | Excessively Deep Inheritance | Flatten; replace the chain with composition or strategy |
| 1075 | Unconditional Control Flow Transfer outside Switch | Remove the statement or the unreachable code after it |
| 1079 | Parent Class without Virtual Destructor | `virtual ~Base() = default;` |
| 1080 | File with Excessive Lines of Code | Split along the responsibility seams already visible |
| 1084 | Excessive File or Data Access Operations | Batch the queries; move I/O out of business logic |
| 1085 | Excessive Volume of Commented-out Code | Delete it; leave a one-line note if the reason matters |
| 1086 | Class with Excessive Number of Child Classes | Collapse data-only subclasses into one parameterised class |
| 1087 | Class with Virtual Method without Virtual Destructor | Add the virtual destructor the polymorphism implies |
| 1090 | Access of a Member Element from Another Class | Ask the owning object; do not reach inside it |
| 1095 | Loop Condition Value Update within the Loop | One owner of the index; use `while` if the stride varies |
| 1121 | Excessive McCabe Cyclomatic Complexity | Extract branches into named predicates or a dispatch table |
