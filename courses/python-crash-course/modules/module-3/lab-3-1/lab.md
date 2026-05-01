# Object-Oriented Python — Classes and Inheritance

In this lab you will learn how to model real-world entities as classes, share behaviour through inheritance, and write code that is easy to extend without modification. You will build a small invoicing system — a realistic domain where OOP pays clear dividends.

**Prerequisites:** Lab 2-2 completed. `python-lab/` directory exists.

---

## Step 1 — Defining a Class

A class is a blueprint for creating objects. It bundles data (attributes) and behaviour (methods) together. `__init__` is the initialiser — it runs automatically when you create a new instance.

```python
# invoicing.py  (create inside python-lab/)

class Product:
    """A product that can appear on an invoice line."""

    def __init__(self, name, unit_price):
        self.name = name
        self.unit_price = unit_price   # price per unit in cents (avoid float rounding)

    def __repr__(self):
        return f"Product({self.name!r}, {self.unit_price / 100:.2f})"

    def __str__(self):
        return f"{self.name} (${self.unit_price / 100:.2f})"


# Creating instances
apple = Product("Apple", 150)       # $1.50
laptop = Product("Laptop", 129999)  # $1,299.99

print(apple)        # Apple ($1.50)
print(repr(apple))  # Product('Apple', 1.50)
print(laptop.name, laptop.unit_price)
```

```bash
cd python-lab
python3 invoicing.py
```

> **`__repr__` vs `__str__`:** `__repr__` is for developers — it should ideally be a string you could paste into the REPL to recreate the object. `__str__` is for end users. When only `__repr__` is defined, Python uses it for both.

---

## Step 2 — Instance Methods and Properties

Methods are functions defined inside a class. They always receive the instance as their first argument (conventionally named `self`). A `@property` turns a method into an attribute you can read without calling it.

```python
# Add to invoicing.py

class LineItem:
    """One line on an invoice: a product and a quantity."""

    def __init__(self, product, quantity):
        if quantity <= 0:
            raise ValueError(f"Quantity must be positive, got {quantity}")
        self.product = product
        self.quantity = quantity

    @property
    def total(self):
        return self.product.unit_price * self.quantity

    def __str__(self):
        return (
            f"{self.product.name:20} x{self.quantity:3}  "
            f"${self.total / 100:>8.2f}"
        )


item1 = LineItem(apple, 4)
item2 = LineItem(laptop, 2)

print(item1)            # Apple                x  4     $6.00
print(item2)
print(item1.total)      # 600  (cents)
```

```bash
python3 invoicing.py
```

> **Store money as integers (cents/pence)** rather than floats. Floating-point arithmetic introduces rounding errors: `0.1 + 0.2 == 0.30000000000000004` in Python (and every language using IEEE 754). Convert to a decimal only when displaying.

---

## Step 3 — Building the Invoice Class

Compose `LineItem` objects inside an `Invoice` class to see how objects work together.

```python
# Add to invoicing.py
from datetime import date

class Invoice:
    """An invoice containing one or more line items."""

    def __init__(self, invoice_number, customer_name):
        self.invoice_number = invoice_number
        self.customer_name = customer_name
        self.date = date.today()
        self._items = []   # _ prefix signals "internal — don't access directly"

    def add_item(self, product, quantity):
        self._items.append(LineItem(product, quantity))

    @property
    def subtotal(self):
        return sum(item.total for item in self._items)

    @property
    def tax(self):
        return int(self.subtotal * 0.10)   # 10% tax, rounded down

    @property
    def total(self):
        return self.subtotal + self.tax

    def __str__(self):
        lines = [
            f"Invoice #{self.invoice_number}",
            f"Customer: {self.customer_name}",
            f"Date:     {self.date}",
            "-" * 40,
        ]
        for item in self._items:
            lines.append(str(item))
        lines += [
            "-" * 40,
            f"{'Subtotal':30} ${self.subtotal / 100:>8.2f}",
            f"{'Tax (10%)':30} ${self.tax / 100:>8.2f}",
            f"{'TOTAL':30} ${self.total / 100:>8.2f}",
        ]
        return "\n".join(lines)


inv = Invoice("2026-001", "Acme Corp")
inv.add_item(apple, 10)
inv.add_item(laptop, 2)
print(inv)
```

```bash
python3 invoicing.py
```

---

## Step 4 — Class Methods and Static Methods

A `@classmethod` receives the class itself (not an instance) as its first argument. It is typically used as an alternative constructor. A `@staticmethod` is just a plain function namespaced inside the class — it receives neither instance nor class.

```python
# Add to invoicing.py

class Invoice:
    _counter = 0   # class variable — shared across all instances

    def __init__(self, invoice_number, customer_name):
        Invoice._counter += 1
        self.invoice_number = invoice_number
        self.customer_name = customer_name
        self.date = date.today()
        self._items = []

    @classmethod
    def create(cls, customer_name):
        """Alternative constructor that auto-generates the invoice number."""
        number = f"{date.today().year}-{cls._counter + 1:04d}"
        return cls(number, customer_name)

    @staticmethod
    def format_currency(cents):
        """Convert integer cents to a formatted string."""
        return f"${cents / 100:,.2f}"

    def add_item(self, product, quantity):
        self._items.append(LineItem(product, quantity))

    @property
    def subtotal(self):
        return sum(item.total for item in self._items)

    @property
    def tax(self):
        return int(self.subtotal * 0.10)

    @property
    def total(self):
        return self.subtotal + self.tax

    def __str__(self):
        lines = [
            f"Invoice #{self.invoice_number}",
            f"Customer: {self.customer_name}",
            f"Date:     {self.date}",
            "-" * 40,
        ]
        for item in self._items:
            lines.append(str(item))
        lines += [
            "-" * 40,
            f"{'Subtotal':30} {Invoice.format_currency(self.subtotal):>10}",
            f"{'Tax (10%)':30} {Invoice.format_currency(self.tax):>10}",
            f"{'TOTAL':30} {Invoice.format_currency(self.total):>10}",
        ]
        return "\n".join(lines)


inv1 = Invoice.create("Acme Corp")
inv1.add_item(apple, 10)
inv1.add_item(laptop, 2)

inv2 = Invoice.create("Globex Ltd")
inv2.add_item(Product("Monitor", 39999), 3)

print(inv1)
print()
print(inv2)
print(f"\nTotal invoices issued: {Invoice._counter}")
```

```bash
python3 invoicing.py
```

---

## Step 5 — Inheritance

Inheritance lets a child class reuse and extend the behaviour of a parent class. Use it when you have a genuine "is-a" relationship — not just because two classes share some code.

```python
# Add to invoicing.py

class DiscountedInvoice(Invoice):
    """An invoice with a percentage discount applied before tax."""

    def __init__(self, customer_name, discount_pct):
        super().__init__(Invoice.create(customer_name).invoice_number, customer_name)
        if not 0 < discount_pct <= 100:
            raise ValueError("Discount must be between 0 and 100")
        self.discount_pct = discount_pct

    @property
    def discount(self):
        raw_subtotal = sum(item.total for item in self._items)
        return int(raw_subtotal * self.discount_pct / 100)

    @property
    def subtotal(self):
        return sum(item.total for item in self._items) - self.discount

    def __str__(self):
        base = super().__str__()
        note = f"  ({self.discount_pct}% discount applied: -{Invoice.format_currency(self.discount)})"
        lines = base.splitlines()
        lines.insert(3, note)
        return "\n".join(lines)


promo_inv = DiscountedInvoice("Big Client", discount_pct=15)
promo_inv.add_item(laptop, 5)
promo_inv.add_item(Product("Mouse", 2999), 5)
print(promo_inv)
```

```bash
python3 invoicing.py
```

> **`super()`** calls the parent class's version of a method. Always use it in `__init__` when inheriting, so the parent's initialisation runs before you add your own setup. Skipping `super().__init__()` is one of the most common inheritance bugs.

---

## Step 6 — Putting It Together: Export to JSON

Add a method to the base `Invoice` class that serialises an invoice to a JSON-ready dict — a pattern you will use constantly when building APIs or saving data.

```python
# Add to the Invoice class definition

    def to_dict(self):
        return {
            "invoice_number": self.invoice_number,
            "customer_name": self.customer_name,
            "date": self.date.isoformat(),
            "items": [
                {
                    "product": item.product.name,
                    "unit_price": item.product.unit_price,
                    "quantity": item.quantity,
                    "total": item.total,
                }
                for item in self._items
            ],
            "subtotal": self.subtotal,
            "tax": self.tax,
            "total": self.total,
        }

# At the bottom of the file
import json
from pathlib import Path

invoices = [inv1, inv2]
data = [inv.to_dict() for inv in invoices]

output = Path("invoices.json")
with output.open("w") as f:
    json.dump(data, f, indent=2)

print(f"Exported {len(data)} invoices to {output}")
print(output.read_text())
```

```bash
python3 invoicing.py
```

You have built a multi-class invoicing system using the full OOP toolkit: instance methods, properties, class methods, static methods, and inheritance. The final lab covers error handling, modules, and virtual environments — the last pieces you need to ship Python code professionally.
