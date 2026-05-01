# Error Handling, Modules, and Virtual Environments

In this lab you will learn how to make your Python code robust and shareable. You will define custom exceptions, use `try/except/finally` correctly, split code into importable modules, and set up a virtual environment with a `requirements.txt` — the standard way to share a Python project so that anyone can reproduce your exact setup.

**Prerequisites:** Lab 3-1 completed. `python-lab/invoicing.py` exists.

---

## Step 1 — Exception Basics

Python signals errors through exceptions. When an exception is raised and not caught, the program prints a traceback and exits. Catching exceptions with `try/except` lets you recover gracefully.

```python
# exceptions_demo.py  (create inside python-lab/)

# Basic try/except
try:
    result = 10 / 0
except ZeroDivisionError:
    print("Cannot divide by zero")

# Catching multiple exception types
def parse_age(value):
    try:
        age = int(value)
        if age < 0 or age > 150:
            raise ValueError(f"Age {age} is out of plausible range")
        return age
    except (ValueError, TypeError) as e:
        print(f"Invalid age: {e}")
        return None

print(parse_age("25"))      # 25
print(parse_age("abc"))     # Invalid age: ...
print(parse_age(None))      # Invalid age: ...
print(parse_age("200"))     # Invalid age: Age 200 is out of plausible range

# else and finally
def read_number(filename):
    try:
        with open(filename) as f:
            return int(f.read().strip())
    except FileNotFoundError:
        print(f"File not found: {filename}")
    except ValueError:
        print(f"File does not contain an integer: {filename}")
    else:
        print("File read successfully")   # only runs if no exception was raised
    finally:
        print("read_number finished")     # always runs, even on exception

read_number("missing.txt")
```

```bash
cd python-lab
python3 exceptions_demo.py
```

| Clause | When it runs |
|--------|-------------|
| `except` | When the matching exception is raised in `try` |
| `else` | When `try` completes with **no** exception |
| `finally` | **Always** — whether or not an exception occurred |

---

## Step 2 — Custom Exceptions

Define your own exception classes for errors that are specific to your domain. This makes calling code clearer and lets callers catch your errors independently of built-in ones.

```python
# Add to exceptions_demo.py


class InvoiceError(Exception):
    """Base class for all invoicing errors."""


class InvoiceNotFoundError(InvoiceError):
    """Raised when a requested invoice does not exist."""

    def __init__(self, invoice_number):
        self.invoice_number = invoice_number
        super().__init__(f"Invoice {invoice_number} not found")


class InvalidLineItemError(InvoiceError):
    """Raised when a line item has an invalid quantity or price."""

    def __init__(self, product_name, reason):
        super().__init__(f"Invalid line item for '{product_name}': {reason}")


# Using custom exceptions
def get_invoice(invoice_db, number):
    if number not in invoice_db:
        raise InvoiceNotFoundError(number)
    return invoice_db[number]


invoice_db = {"2026-0001": {"customer": "Acme", "total": 5000}}

try:
    inv = get_invoice(invoice_db, "2026-9999")
except InvoiceNotFoundError as e:
    print(f"Lookup failed: {e}")
    print(f"Number was: {e.invoice_number}")
except InvoiceError as e:
    print(f"General invoice error: {e}")
```

```bash
python3 exceptions_demo.py
```

> **Exception hierarchy:** Catch the most specific exception first, then broader ones. Python matches the first `except` clause that fits, so a broad `except Exception` before a specific one would shadow it.

---

## Step 3 — Organising Code into Modules

As programs grow, a single file becomes unwieldy. Python's module system lets you split code across files and import what you need. Any `.py` file is a module; a directory with an `__init__.py` is a package.

```bash
# Create a package structure inside python-lab/
mkdir -p invoicing_app
```

```python
# invoicing_app/__init__.py  (empty — marks the directory as a package)
```

```python
# invoicing_app/exceptions.py

class InvoiceError(Exception):
    """Base class for all invoicing errors."""

class InvoiceNotFoundError(InvoiceError):
    def __init__(self, invoice_number):
        self.invoice_number = invoice_number
        super().__init__(f"Invoice {invoice_number} not found")

class InvalidLineItemError(InvoiceError):
    def __init__(self, product_name, reason):
        super().__init__(f"Invalid line item for '{product_name}': {reason}")
```

```python
# invoicing_app/models.py

from datetime import date
from .exceptions import InvalidLineItemError


class Product:
    def __init__(self, name, unit_price):
        if unit_price < 0:
            raise InvalidLineItemError(name, "unit price cannot be negative")
        self.name = name
        self.unit_price = unit_price

    def __repr__(self):
        return f"Product({self.name!r}, {self.unit_price})"


class LineItem:
    def __init__(self, product, quantity):
        if quantity <= 0:
            raise InvalidLineItemError(product.name, f"quantity must be positive, got {quantity}")
        self.product = product
        self.quantity = quantity

    @property
    def total(self):
        return self.product.unit_price * self.quantity
```

```python
# invoicing_app/invoice.py

from datetime import date
from .models import LineItem


class Invoice:
    _counter = 0

    def __init__(self, customer_name):
        Invoice._counter += 1
        self.invoice_number = f"{date.today().year}-{Invoice._counter:04d}"
        self.customer_name = customer_name
        self.date = date.today()
        self._items = []

    def add_item(self, product, quantity):
        self._items.append(LineItem(product, quantity))

    @property
    def total(self):
        return sum(item.total for item in self._items)

    def to_dict(self):
        return {
            "invoice_number": self.invoice_number,
            "customer": self.customer_name,
            "date": self.date.isoformat(),
            "total": self.total,
        }
```

```python
# main.py  (in python-lab/ — the entry point)

from invoicing_app.models import Product
from invoicing_app.invoice import Invoice
from invoicing_app.exceptions import InvoiceError

laptop = Product("Laptop", 129999)
mouse  = Product("Mouse", 2999)

inv = Invoice("Acme Corp")
inv.add_item(laptop, 2)
inv.add_item(mouse, 4)

print(f"Invoice {inv.invoice_number}: total = ${inv.total / 100:,.2f}")
```

```bash
python3 main.py
```

> **Relative imports** (`from .models import LineItem`) work inside a package. Absolute imports (`from invoicing_app.models import LineItem`) work from outside the package. Use absolute imports in entry-point scripts (`main.py`).

---

## Step 4 — Virtual Environments

A virtual environment is an isolated Python installation for a single project. It prevents packages installed for one project from breaking another. Every Python project should have one.

```bash
# Create a virtual environment (run from python-lab/)
python3 -m venv .venv

# Activate it
# On Linux / macOS:
source .venv/bin/activate
# On Windows Git Bash:
source .venv/Scripts/activate

# Your prompt now shows (.venv) — all pip commands use this environment
(.venv) $ python --version
(.venv) $ pip --version

# Install a package
(.venv) $ pip install requests

# See what is installed
(.venv) $ pip list

# Deactivate when done
(.venv) $ deactivate
```

Add the venv to `.gitignore` so it is never committed:

```bash
echo ".venv/" >> .gitignore
echo "__pycache__/" >> .gitignore
echo "*.pyc" >> .gitignore
```

> **Never commit `.venv/`**. It contains compiled binaries specific to your OS and Python version. Share a `requirements.txt` instead (next step) so others can recreate the environment.

---

## Step 5 — requirements.txt and Reproducible Environments

`requirements.txt` is the conventional file for listing your project's dependencies. Anyone who clones your project can run one command to install exactly what you used.

```bash
# Activate your venv first, then install a few packages
source .venv/bin/activate
pip install requests rich

# Pin your current dependencies to requirements.txt
pip freeze > requirements.txt

cat requirements.txt
```

The file will look something like:
```
certifi==2024.12.14
charset-normalizer==3.4.0
idna==3.10
requests==2.32.3
rich==13.9.4
...
```

Recreating the environment on another machine:
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Create a minimal `requirements.txt` for this course project (without pinned sub-dependencies):

```bash
cat > requirements.txt << 'EOF'
requests>=2.28
rich>=13.0
EOF
```

> **Pin in production, range in libraries.** Application deployments (servers, scripts) should use exact pins (`==`) from `pip freeze` for reproducibility. Library packages should use minimum version constraints (`>=`) so they don't conflict with their users' other dependencies.

---

## Step 6 — Using a Third-Party Library: Pretty Output with `rich`

Apply what you have learned by using `rich` — a popular library for beautiful terminal output — to render an invoice table. This is the complete workflow: install a package, import it, use it.

```python
# pretty_invoice.py  (in python-lab/)

from rich.console import Console
from rich.table import Table
from rich import box

from invoicing_app.models import Product
from invoicing_app.invoice import Invoice

console = Console()

laptop  = Product("Laptop",  129999)
monitor = Product("Monitor",  39999)
mouse   = Product("Mouse",    2999)
keyboard = Product("Keyboard", 7999)

inv = Invoice("Globex Ltd")
inv.add_item(laptop, 2)
inv.add_item(monitor, 3)
inv.add_item(mouse, 5)
inv.add_item(keyboard, 5)

table = Table(title=f"Invoice #{inv.invoice_number}", box=box.ROUNDED)
table.add_column("Product",    style="cyan",  no_wrap=True)
table.add_column("Unit Price", justify="right")
table.add_column("Qty",        justify="right")
table.add_column("Line Total", justify="right", style="green")

for item in inv._items:
    table.add_row(
        item.product.name,
        f"${item.product.unit_price / 100:,.2f}",
        str(item.quantity),
        f"${item.total / 100:,.2f}",
    )

console.print(table)
console.print(f"\n[bold]Customer:[/bold] {inv.customer_name}")
console.print(f"[bold]Total:   [/bold] [green]${inv.total / 100:,.2f}[/green]")
```

```bash
# Make sure your venv is active and rich is installed
python3 pretty_invoice.py
```

You have now covered everything you need to write, organise, and share professional Python code: robust error handling with custom exceptions, packages with relative imports, virtual environments, and third-party libraries via pip. The patterns in this course — functions, classes, file I/O, modules, and venvs — are the foundation every Python developer builds on every day.
