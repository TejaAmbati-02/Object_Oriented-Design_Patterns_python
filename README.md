# Object-Oriented Design Patterns

A focused guide to 10 essential design patterns with Python examples.

---

## Table of Contents

**Creational**
- [Factory](#factory)
- [Singleton](#singleton)
- [Builder](#builder)
- [Prototype](#prototype)

**Structural**
- [Adapter](#adapter)
- [Decorator](#decorator)
- [Facade](#facade)

**Behavioral**
- [Strategy](#strategy)
- [Observer](#observer)
- [State](#state)

---

## Creational Patterns

> Deal with **object creation** — controlling how and when objects are instantiated.

---

### Factory

**Intent:** Define an interface for creating an object, but let subclasses decide which class to instantiate. Removes the need for `if/else` chains when creating objects.

**When to use:**
- You don't know ahead of time which class you need to instantiate.
- You want to centralize and encapsulate object creation logic.

```python
from abc import ABC, abstractmethod

class Notification(ABC):
    @abstractmethod
    def send(self, message: str) -> str:
        pass

class EmailNotification(Notification):
    def send(self, message: str) -> str:
        return f"[Email] {message}"

class SMSNotification(Notification):
    def send(self, message: str) -> str:
        return f"[SMS] {message}"

class PushNotification(Notification):
    def send(self, message: str) -> str:
        return f"[Push] {message}"

class NotificationFactory:
    _registry = {
        "email": EmailNotification,
        "sms": SMSNotification,
        "push": PushNotification,
    }

    @classmethod
    def create(cls, channel: str) -> Notification:
        if channel not in cls._registry:
            raise ValueError(f"Unknown channel: {channel}")
        return cls._registry[channel]()

# Client code never uses `new` (or constructors) directly
for channel in ["email", "sms", "push"]:
    notif = NotificationFactory.create(channel)
    print(notif.send("Welcome aboard!"))

# [Email] Welcome aboard!
# [SMS] Welcome aboard!
# [Push] Welcome aboard!
```

**Pros:** Loose coupling, easy to add new types without changing client code.  
**Cons:** Requires a new class per product type.

---

### Singleton

**Intent:** Ensure a class has only **one instance** and provide a global point of access to it.

**When to use:**
- Shared resources: config objects, loggers, database connection pools.
- You need strict control over a single shared state.

```python
class AppConfig:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._settings = {}
            print("  Config instance created")
        return cls._instance

    def set(self, key: str, value):
        self._settings[key] = value

    def get(self, key: str):
        return self._settings.get(key)

# First call — creates the instance
config1 = AppConfig()
config1.set("theme", "dark")

# Second call — returns the same instance
config2 = AppConfig()
print(config2.get("theme"))  # dark
print(config1 is config2)    # True — same object
```

**Pros:** Controlled single access point, lazy initialization.  
**Cons:** Global state makes unit testing harder; use with care.

---

### Builder

**Intent:** Separate the **construction** of a complex object from its **representation**. Build an object step by step using method chaining.

**When to use:**
- Objects require many optional parameters or configuration steps.
- You want to avoid telescoping constructors (`__init__` with 10 arguments).

```python
class QueryBuilder:
    def __init__(self, table: str):
        self._table = table
        self._conditions: list[str] = []
        self._columns: list[str] = ["*"]
        self._limit: int | None = None
        self._order_by: str | None = None

    def select(self, *columns: str) -> "QueryBuilder":
        self._columns = list(columns)
        return self

    def where(self, condition: str) -> "QueryBuilder":
        self._conditions.append(condition)
        return self

    def order_by(self, column: str) -> "QueryBuilder":
        self._order_by = column
        return self

    def limit(self, n: int) -> "QueryBuilder":
        self._limit = n
        return self

    def build(self) -> str:
        query = f"SELECT {', '.join(self._columns)} FROM {self._table}"
        if self._conditions:
            query += " WHERE " + " AND ".join(self._conditions)
        if self._order_by:
            query += f" ORDER BY {self._order_by}"
        if self._limit is not None:
            query += f" LIMIT {self._limit}"
        return query

query = (
    QueryBuilder("users")
    .select("id", "name", "email")
    .where("age > 18")
    .where("active = true")
    .order_by("name")
    .limit(10)
    .build()
)
print(query)
# SELECT id, name, email FROM users WHERE age > 18 AND active = true ORDER BY name LIMIT 10
```

**Pros:** Readable construction, supports optional fields cleanly.  
**Cons:** More boilerplate for simple objects.

---

### Prototype

**Intent:** Create new objects by **cloning** an existing instance rather than constructing from scratch.

**When to use:**
- Object creation is expensive (e.g., database queries, network calls).
- You need many similar objects with small variations.

```python
import copy

class Enemy:
    def __init__(self, name: str, hp: int, damage: int, abilities: list[str]):
        self.name = name
        self.hp = hp
        self.damage = damage
        self.abilities = abilities

    def clone(self) -> "Enemy":
        return copy.deepcopy(self)

    def __str__(self):
        return f"{self.name} | HP: {self.hp} | DMG: {self.damage} | Abilities: {self.abilities}"

# Define a base template
base_goblin = Enemy("Goblin", hp=50, damage=10, abilities=["bite"])

# Spawn variants by cloning — no expensive re-initialization
goblin1 = base_goblin.clone()

goblin2 = base_goblin.clone()
goblin2.name = "Goblin Archer"
goblin2.damage = 20
goblin2.abilities.append("arrow shot")  # deepcopy keeps this isolated

print(base_goblin)  # Goblin | HP: 50 | DMG: 10 | Abilities: ['bite']
print(goblin1)      # Goblin | HP: 50 | DMG: 10 | Abilities: ['bite']
print(goblin2)      # Goblin Archer | HP: 50 | DMG: 20 | Abilities: ['bite', 'arrow shot']
```

**Pros:** Avoids costly re-initialization, easy to create variants.  
**Cons:** Deep-copying complex objects (with circular refs) can be tricky.

---

## Structural Patterns

> Deal with **object composition** — how classes and objects are combined to form larger structures.

---

### Adapter

**Intent:** Convert the interface of an existing class into an interface the client expects. Makes **incompatible interfaces** work together.

**When to use:**
- Integrating a third-party library or legacy code whose interface doesn't match yours.
- You can't change the source class but need it to fit your system.

```python
# Third-party payment processor — interface you cannot change
class StripeAPI:
    def make_charge(self, amount_cents: int, currency: str) -> dict:
        return {
            "status": "charged",
            "amount": amount_cents,
            "currency": currency,
        }

# Your app's payment interface
class PaymentProcessor:
    def pay(self, amount_dollars: float) -> str:
        raise NotImplementedError

# Adapter bridges your interface with Stripe's interface
class StripeAdapter(PaymentProcessor):
    def __init__(self):
        self._stripe = StripeAPI()

    def pay(self, amount_dollars: float) -> str:
        cents = int(amount_dollars * 100)
        result = self._stripe.make_charge(cents, "USD")
        return f"Payment {result['status']}: ${amount_dollars:.2f}"

# Client only knows about PaymentProcessor — not Stripe internals
def checkout(processor: PaymentProcessor, amount: float):
    print(processor.pay(amount))

checkout(StripeAdapter(), 29.99)  # Payment charged: $29.99
```

**Pros:** Reuses existing code without modification, respects Open/Closed Principle.  
**Cons:** Adds an extra layer of indirection.

---

### Decorator

**Intent:** Attach additional behavior to an object **dynamically** by wrapping it. A flexible alternative to subclassing.

**When to use:**
- You need to add features to individual objects, not all instances of a class.
- Using inheritance would create an explosion of subclass combinations.

```python
from abc import ABC, abstractmethod

class TextFormatter(ABC):
    @abstractmethod
    def format(self, text: str) -> str:
        pass

class PlainText(TextFormatter):
    def format(self, text: str) -> str:
        return text

class BoldDecorator(TextFormatter):
    def __init__(self, wrapped: TextFormatter):
        self._wrapped = wrapped

    def format(self, text: str) -> str:
        return f"**{self._wrapped.format(text)}**"

class ItalicDecorator(TextFormatter):
    def __init__(self, wrapped: TextFormatter):
        self._wrapped = wrapped

    def format(self, text: str) -> str:
        return f"_{self._wrapped.format(text)}_"

class UpperCaseDecorator(TextFormatter):
    def __init__(self, wrapped: TextFormatter):
        self._wrapped = wrapped

    def format(self, text: str) -> str:
        return self._wrapped.format(text).upper()

text = "hello world"

plain = PlainText()
bold = BoldDecorator(PlainText())
bold_italic = ItalicDecorator(BoldDecorator(PlainText()))
all_three = UpperCaseDecorator(ItalicDecorator(BoldDecorator(PlainText())))

print(plain.format(text))      # hello world
print(bold.format(text))       # **hello world**
print(bold_italic.format(text)) # _**hello world**_
print(all_three.format(text))  # _**HELLO WORLD**_
```

**Pros:** Composable at runtime, avoids class explosion from inheritance.  
**Cons:** Many small wrapper objects can be hard to debug.

---

### Facade

**Intent:** Provide a **simple, unified interface** to a complex subsystem.

**When to use:**
- A subsystem has many moving parts and clients only need a high-level interface.
- You want to decouple clients from internal implementation details.

```python
# Complex subsystem — each class does one thing
class VideoDecoder:
    def decode(self, filename: str) -> str:
        print(f"  VideoDecoder: decoding '{filename}'")
        return "raw_frames"

class AudioDecoder:
    def decode(self, filename: str) -> str:
        print(f"  AudioDecoder: decoding '{filename}'")
        return "raw_audio"

class AudioMixer:
    def normalize(self, audio: str) -> str:
        print(f"  AudioMixer: normalizing audio")
        return "normalized_audio"

class VideoRenderer:
    def render(self, frames: str, audio: str) -> str:
        print(f"  VideoRenderer: combining frames + audio")
        return "final_video.mp4"

class Uploader:
    def upload(self, video: str, destination: str):
        print(f"  Uploader: uploading '{video}' to {destination}")

# Facade — one method hides the entire pipeline
class VideoProcessingFacade:
    def __init__(self):
        self._vdecoder = VideoDecoder()
        self._adecoder = AudioDecoder()
        self._mixer = AudioMixer()
        self._renderer = VideoRenderer()
        self._uploader = Uploader()

    def process_and_upload(self, filename: str, destination: str):
        print(f"Processing '{filename}'...")
        frames = self._vdecoder.decode(filename)
        raw_audio = self._adecoder.decode(filename)
        clean_audio = self._mixer.normalize(raw_audio)
        output = self._renderer.render(frames, clean_audio)
        self._uploader.upload(output, destination)
        print(f"Done. Video available at {destination}\n")

facade = VideoProcessingFacade()
facade.process_and_upload("lecture.mp4", "cdn.example.com/videos")
```

**Pros:** Simplifies client code, decouples clients from subsystem internals.  
**Cons:** Facade can become a bloated "god object" if not kept focused.

---

## Behavioral Patterns

> Deal with **communication between objects** — how responsibility is distributed and algorithms are encapsulated.

---

### Strategy

**Intent:** Define a family of algorithms, encapsulate each one, and make them **interchangeable** at runtime.

**When to use:**
- You need to switch between different variants of an algorithm.
- You want to eliminate `if/elif` chains that choose behavior.

```python
from abc import ABC, abstractmethod

class ShippingStrategy(ABC):
    @abstractmethod
    def calculate(self, weight_kg: float) -> float:
        pass

    @abstractmethod
    def name(self) -> str:
        pass

class StandardShipping(ShippingStrategy):
    def calculate(self, weight_kg: float) -> float:
        return 5.00 + (weight_kg * 1.50)

    def name(self) -> str:
        return "Standard (5-7 days)"

class ExpressShipping(ShippingStrategy):
    def calculate(self, weight_kg: float) -> float:
        return 15.00 + (weight_kg * 3.00)

    def name(self) -> str:
        return "Express (1-2 days)"

class FreeShipping(ShippingStrategy):
    def calculate(self, weight_kg: float) -> float:
        return 0.00

    def name(self) -> str:
        return "Free Shipping (7-14 days)"

class ShoppingCart:
    def __init__(self):
        self._items: list[tuple[str, float]] = []
        self._strategy: ShippingStrategy = StandardShipping()

    def add_item(self, name: str, weight_kg: float):
        self._items.append((name, weight_kg))

    def set_shipping(self, strategy: ShippingStrategy):
        self._strategy = strategy

    def checkout(self):
        total_weight = sum(w for _, w in self._items)
        shipping_cost = self._strategy.calculate(total_weight)
        print(f"Shipping: {self._strategy.name()} — ${shipping_cost:.2f}")

cart = ShoppingCart()
cart.add_item("Laptop", 2.5)
cart.add_item("Book", 0.5)

cart.set_shipping(StandardShipping())
cart.checkout()  # Shipping: Standard (5-7 days) — $9.50

cart.set_shipping(ExpressShipping())
cart.checkout()  # Shipping: Express (1-2 days) — $24.00

cart.set_shipping(FreeShipping())
cart.checkout()  # Shipping: Free Shipping (7-14 days) — $0.00
```

**Pros:** Eliminates conditionals, algorithms are swappable at runtime.  
**Cons:** Client must know which strategies exist and when to use them.

---

### Observer

**Intent:** Define a **one-to-many** dependency so that when one object changes state, all dependents are notified automatically.

**When to use:**
- Event systems, pub/sub messaging.
- UI updates when model data changes (MVC).
- Any "when X happens, do Y and Z" scenario.

```python
from abc import ABC, abstractmethod

class Observer(ABC):
    @abstractmethod
    def update(self, event: str, data):
        pass

class EventBus:
    def __init__(self):
        self._subscribers: dict[str, list[Observer]] = {}

    def subscribe(self, event: str, observer: Observer):
        self._subscribers.setdefault(event, []).append(observer)

    def unsubscribe(self, event: str, observer: Observer):
        self._subscribers.get(event, []).remove(observer)

    def publish(self, event: str, data=None):
        for observer in self._subscribers.get(event, []):
            observer.update(event, data)

class EmailService(Observer):
    def update(self, event: str, data):
        print(f"  [EmailService] Sending email for '{event}': {data}")

class InventoryService(Observer):
    def update(self, event: str, data):
        print(f"  [InventoryService] Updating stock for '{event}': {data}")

class AnalyticsService(Observer):
    def update(self, event: str, data):
        print(f"  [AnalyticsService] Recording '{event}': {data}")

bus = EventBus()
email = EmailService()
inventory = InventoryService()
analytics = AnalyticsService()

bus.subscribe("order_placed", email)
bus.subscribe("order_placed", inventory)
bus.subscribe("order_placed", analytics)
bus.subscribe("user_signup", email)

print("=== Order placed ===")
bus.publish("order_placed", {"item": "Laptop", "qty": 1})

print("\n=== User signed up ===")
bus.publish("user_signup", {"user": "alice@example.com"})
```

**Pros:** Loose coupling between publisher and subscribers, easy to add new observers.  
**Cons:** Memory leaks if observers aren't unsubscribed; unexpected cascade updates.

---

### State

**Intent:** Allow an object to **alter its behavior** when its internal state changes. The object appears to change its class.

**When to use:**
- An object's behavior depends on its current state (e.g., order status, traffic lights, vending machines).
- You want to replace large `if/elif` blocks based on state flags.

```python
from abc import ABC, abstractmethod

class State(ABC):
    @abstractmethod
    def insert_coin(self, machine: "VendingMachine"):
        pass

    @abstractmethod
    def select_item(self, machine: "VendingMachine"):
        pass

    @abstractmethod
    def dispense(self, machine: "VendingMachine"):
        pass

    def __str__(self) -> str:
        return self.__class__.__name__

class IdleState(State):
    def insert_coin(self, machine: "VendingMachine"):
        print("  Coin accepted.")
        machine.state = HasCoinState()

    def select_item(self, machine: "VendingMachine"):
        print("  Please insert a coin first.")

    def dispense(self, machine: "VendingMachine"):
        print("  Please insert a coin first.")

class HasCoinState(State):
    def insert_coin(self, machine: "VendingMachine"):
        print("  Coin already inserted. Returning extra coin.")

    def select_item(self, machine: "VendingMachine"):
        print("  Item selected.")
        machine.state = DispensingState()

    def dispense(self, machine: "VendingMachine"):
        print("  Please select an item first.")

class DispensingState(State):
    def insert_coin(self, machine: "VendingMachine"):
        print("  Please wait, dispensing in progress.")

    def select_item(self, machine: "VendingMachine"):
        print("  Please wait, dispensing in progress.")

    def dispense(self, machine: "VendingMachine"):
        print("  Dispensing item... Enjoy!")
        machine.state = IdleState()

class VendingMachine:
    def __init__(self):
        self.state: State = IdleState()

    def insert_coin(self): self.state.insert_coin(self)
    def select_item(self): self.state.select_item(self)
    def dispense(self): self.state.dispense(self)

    def status(self): print(f"[State: {self.state}]")

machine = VendingMachine()
machine.status()          # [State: IdleState]

machine.select_item()     # Please insert a coin first.
machine.insert_coin()     # Coin accepted.
machine.status()          # [State: HasCoinState]

machine.insert_coin()     # Coin already inserted.
machine.select_item()     # Item selected.
machine.status()          # [State: DispensingState]

machine.dispense()        # Dispensing item... Enjoy!
machine.status()          # [State: IdleState]
```

**Pros:** Eliminates state-based conditionals, each state is self-contained and easy to test.  
**Cons:** Overkill for just 2–3 states; adds many classes for complex state machines.

---

## Quick Reference

| Category    | Pattern    | One-Line Summary                                     |
|-------------|------------|------------------------------------------------------|
| Creational  | Factory    | Centralize object creation behind a method           |
| Creational  | Singleton  | One shared instance, globally accessible             |
| Creational  | Builder    | Construct complex objects step by step               |
| Creational  | Prototype  | Clone an existing object instead of building anew    |
| Structural  | Adapter    | Wrap an incompatible interface to fit your own       |
| Structural  | Decorator  | Stack wrappers to add behavior at runtime            |
| Structural  | Facade     | One simple entry point to a complex subsystem        |
| Behavioral  | Strategy   | Swap algorithms in and out at runtime                |
| Behavioral  | Observer   | Notify multiple listeners when state changes         |
| Behavioral  | State      | Let an object change behavior based on internal state|

---

## References

- *Design Patterns: Elements of Reusable Object-Oriented Software* — Gamma, Helm, Johnson, Vlissides (GoF, 1994)
- *Head First Design Patterns* — Freeman & Robson
- [Refactoring.Guru Design Patterns](https://refactoring.guru/design-patterns)
