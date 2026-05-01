# Variables, Types, Control Flow, and Functions

In this lab you will write your first Python script, explore the core data types, control program flow with conditions and loops, and package reusable logic into functions. By the end you will have a small command-line calculator that demonstrates every concept covered.

**Prerequisites:** Python 3.10 or later installed (`python3 --version`). A terminal and a text editor or VS Code.

---

## Step 1 — Your First Python Script

Python code is plain text saved with a `.py` extension. Unlike languages such as Java or C, there are no curly braces — indentation (four spaces) defines blocks. There is also no explicit compile step; you run the file directly with the Python interpreter.

Create your working directory and first script:

```bash
mkdir python-lab
cd python-lab
```

```python
# hello.py
name = "World"
print(f"Hello, {name}!")
```

```bash
python3 hello.py
```

Expected output:
```
Hello, World!
```

> **f-strings** (the `f"..."` syntax) let you embed expressions directly inside string literals. They were introduced in Python 3.6 and are now the standard way to format strings — prefer them over `%` formatting or `.format()`.

---

## Step 2 — Core Data Types

Python has a small set of built-in types that cover most everyday needs. Understanding what each one is for saves you from reinventing the wheel.

```python
# types.py

# Integer and float arithmetic
age = 25
price = 9.99
total = age * price
print(type(total), total)   # <class 'float'> 249.75

# Strings are immutable sequences of characters
greeting = "Hello"
print(greeting.upper())     # HELLO
print(greeting[1:4])        # ell  (slicing)
print(len(greeting))        # 5

# Booleans
is_active = True
is_banned = False
print(is_active and not is_banned)  # True

# None — the absence of a value (similar to null in other languages)
result = None
print(result is None)       # True
```

```bash
python3 types.py
```

| Type | Example | Common operations |
|------|---------|-------------------|
| `int` | `42` | `+`, `-`, `*`, `//` (floor div), `%` (modulo) |
| `float` | `3.14` | same as int, plus `/` for true division |
| `str` | `"hello"` | `+` (concat), `[i]`, `[a:b]`, `.split()`, `.strip()` |
| `bool` | `True` / `False` | `and`, `or`, `not` |
| `None` | `None` | identity check: `x is None` |

---

## Step 3 — Conditions and Comparisons

Python uses `if`, `elif`, and `else` for branching. Comparison operators return booleans that you can combine with `and`, `or`, and `not`.

```python
# conditions.py

score = 72

if score >= 90:
    grade = "A"
elif score >= 80:
    grade = "B"
elif score >= 70:
    grade = "C"
else:
    grade = "F"

print(f"Score {score} → Grade {grade}")

# Membership test — very readable in Python
allowed_roles = {"admin", "editor", "moderator"}
user_role = "editor"

if user_role in allowed_roles:
    print(f"{user_role} has access")
else:
    print(f"{user_role} is not authorised")
```

```bash
python3 conditions.py
```

> **Truthiness:** In Python, empty strings, `0`, `[]`, `{}`, and `None` are all "falsy" — they behave like `False` in a boolean context. Everything else is truthy. This lets you write `if items:` instead of `if len(items) > 0:`.

---

## Step 4 — Loops

Python has two loop forms. `for` iterates over any iterable (list, string, range, dict…). `while` runs as long as a condition is true. Prefer `for` when you know what you are iterating over.

```python
# loops.py

# for loop over a range
for i in range(1, 6):
    print(i, end=" ")   # 1 2 3 4 5
print()

# for loop over a list
fruits = ["apple", "banana", "cherry"]
for fruit in fruits:
    print(fruit.capitalize())

# enumerate — get both index and value
for index, fruit in enumerate(fruits):
    print(f"{index}: {fruit}")

# while loop — use when the stopping condition isn't known in advance
attempts = 0
while attempts < 3:
    print(f"Attempt {attempts + 1}")
    attempts += 1

# Loop control
for n in range(10):
    if n == 5:
        break           # exit the loop entirely
    if n % 2 == 0:
        continue        # skip to next iteration
    print(n)            # prints 1 3
```

```bash
python3 loops.py
```

---

## Step 5 — Functions

Functions let you name and reuse a block of logic. In Python, functions are first-class objects — you can pass them as arguments, return them from other functions, and store them in variables.

```python
# functions.py

def greet(name, greeting="Hello"):
    """Return a personalised greeting string."""
    return f"{greeting}, {name}!"

print(greet("Alice"))               # Hello, Alice!
print(greet("Bob", greeting="Hi")) # Hi, Bob!


def clamp(value, minimum, maximum):
    """Return value constrained to [minimum, maximum]."""
    return max(minimum, min(value, maximum))

print(clamp(150, 0, 100))   # 100
print(clamp(-5, 0, 100))    # 0
print(clamp(42, 0, 100))    # 42


def summarise(*numbers):
    """Accept any number of arguments and return basic stats."""
    total = sum(numbers)
    count = len(numbers)
    return {
        "total": total,
        "count": count,
        "average": total / count if count else 0,
    }

stats = summarise(10, 20, 30, 40)
print(stats)
```

```bash
python3 functions.py
```

> **Docstrings** (the `"""..."""` immediately after `def`) are the standard way to document functions in Python. Tools like `help()`, IDE hover tooltips, and documentation generators all read them automatically.

---

## Step 6 — Putting It Together: A Command-Line Calculator

Combine everything from this lab into a small interactive script. This is the pattern you will use in every Python program: read input, apply logic, print output.

```python
# calculator.py

def add(a, b):
    return a + b

def subtract(a, b):
    return a - b

def multiply(a, b):
    return a * b

def divide(a, b):
    if b == 0:
        return "Error: division by zero"
    return a / b

OPERATIONS = {
    "+": add,
    "-": subtract,
    "*": multiply,
    "/": divide,
}

def calculate(expression):
    """Parse and evaluate a simple 'a op b' expression string."""
    parts = expression.strip().split()
    if len(parts) != 3:
        return "Usage: <number> <operator> <number>"

    a_str, op, b_str = parts

    if op not in OPERATIONS:
        return f"Unknown operator '{op}'. Use one of: {', '.join(OPERATIONS)}"

    try:
        a = float(a_str)
        b = float(b_str)
    except ValueError:
        return "Both operands must be numbers"

    result = OPERATIONS[op](a, b)
    return result

if __name__ == "__main__":
    print("Simple Calculator. Type 'quit' to exit.")
    while True:
        expr = input("> ")
        if expr.lower() in ("quit", "exit", "q"):
            break
        print(calculate(expr))
```

```bash
python3 calculator.py
```

Try:
```
> 10 + 5
15.0
> 100 / 0
Error: division by zero
> quit
```

> **`if __name__ == "__main__":`** is the standard Python idiom for code that should only run when the file is executed directly — not when it is imported as a module by another file. Always use it for scripts that have an entry point.

You now have a working understanding of Python's core syntax: variables, types, conditions, loops, and functions. The next lab builds on these foundations with Python's powerful collection types — lists, dictionaries, sets, and comprehensions.
