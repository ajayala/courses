# Lists, Dicts, Sets, and Comprehensions

In this lab you will master Python's four built-in collection types and learn how to transform data concisely using comprehensions. You will build a small contact book that stores, queries, and filters records — a pattern that appears in almost every real-world Python program.

**Prerequisites:** Lab 1-1 completed. `python-lab/` directory exists.

---

## Step 1 — Lists

A list is an ordered, mutable sequence. It is Python's go-to container when you need to keep items in a specific order or allow duplicates.

```python
# collections_demo.py  (create this file inside python-lab/)

# Creating lists
fruits = ["apple", "banana", "cherry"]
numbers = [10, 20, 30, 40, 50]
mixed = [1, "hello", True, 3.14]   # lists can hold any type

# Indexing and slicing
print(fruits[0])        # apple
print(fruits[-1])       # cherry  (negative index counts from end)
print(numbers[1:4])     # [20, 30, 40]  (slice: start inclusive, end exclusive)

# Mutating a list
fruits.append("date")           # add to end
fruits.insert(1, "avocado")     # insert at position
fruits.remove("banana")         # remove by value
popped = fruits.pop()           # remove and return last item
print(fruits)

# Useful list operations
nums = [3, 1, 4, 1, 5, 9, 2, 6]
print(sorted(nums))             # returns a new sorted list
print(min(nums), max(nums))     # 1 9
print(nums.count(1))            # 2  (how many times 1 appears)
```

```bash
cd python-lab
python3 collections_demo.py
```

> **`sorted()` vs `.sort()`:** `sorted(x)` returns a new list; `x.sort()` sorts in place and returns `None`. When in doubt, use `sorted()` — it never mutates your original data.

---

## Step 2 — Dictionaries

A dictionary maps keys to values. It is the most important collection type in Python — you will use dicts for configuration, API responses, database rows, and almost any record-like structure.

```python
# Add to collections_demo.py

# Creating dicts
contact = {
    "name": "Alice",
    "email": "alice@example.com",
    "age": 30,
}

# Reading values
print(contact["name"])                  # Alice
print(contact.get("phone", "N/A"))      # N/A  (safe: returns default if key missing)

# Modifying
contact["phone"] = "555-1234"           # add or update a key
del contact["age"]                      # delete a key

# Iterating
for key, value in contact.items():
    print(f"  {key}: {value}")

# Checking membership
if "email" in contact:
    print("Has email")

# Dict of dicts — a common pattern for records
contacts = {
    "alice": {"email": "alice@example.com", "city": "London"},
    "bob":   {"email": "bob@example.com",   "city": "Paris"},
}
print(contacts["bob"]["city"])          # Paris
```

```bash
python3 collections_demo.py
```

| Operation | Syntax | Notes |
|-----------|--------|-------|
| Get value (safe) | `d.get(key, default)` | Returns `default` instead of raising `KeyError` |
| All keys | `d.keys()` | Returns a view object |
| All values | `d.values()` | Returns a view object |
| All pairs | `d.items()` | Use in `for k, v in d.items()` |
| Merge dicts | `{**d1, **d2}` | Keys in `d2` overwrite `d1` |

---

## Step 3 — Sets and Tuples

Sets store unique, unordered values — useful for membership tests and removing duplicates. Tuples are immutable sequences — use them when the data should not change (coordinates, config pairs, dict keys).

```python
# Add to collections_demo.py

# Sets — unique values only
tags = {"python", "backend", "api", "python"}   # duplicate "python" is dropped
print(tags)                                      # order is not guaranteed

tags.add("testing")
tags.discard("api")     # remove if present (no error if missing)

# Set operations
backend_skills = {"python", "sql", "docker"}
frontend_skills = {"javascript", "css", "python"}

print(backend_skills & frontend_skills)   # intersection: {"python"}
print(backend_skills | frontend_skills)   # union: all unique skills
print(backend_skills - frontend_skills)   # difference: in backend but not frontend

# Fast membership test — O(1) vs O(n) for lists
STOP_WORDS = {"the", "a", "an", "in", "on", "at"}
words = ["the", "quick", "brown", "fox"]
filtered = [w for w in words if w not in STOP_WORDS]
print(filtered)   # ['quick', 'brown', 'fox']

# Tuples — immutable sequences
point = (10, 20)
x, y = point       # unpacking
print(f"x={x}, y={y}")

rgb = (255, 128, 0)
print(rgb[0])      # 255
# rgb[0] = 0       # ← this would raise TypeError — tuples are immutable
```

```bash
python3 collections_demo.py
```

---

## Step 4 — List Comprehensions

Comprehensions are one of Python's most distinctive features. They let you transform or filter collections in a single, readable expression — replacing many `for` loops that build up a new list.

```python
# comprehensions.py

# Without comprehension — verbose
squares = []
for n in range(1, 6):
    squares.append(n ** 2)
print(squares)   # [1, 4, 9, 16, 25]

# With list comprehension — concise
squares = [n ** 2 for n in range(1, 6)]
print(squares)   # [1, 4, 9, 16, 25]

# With a filter condition
even_squares = [n ** 2 for n in range(1, 11) if n % 2 == 0]
print(even_squares)   # [4, 16, 36, 64, 100]

# Transform strings
words = ["  hello  ", "  WORLD  ", " Python "]
clean = [w.strip().lower() for w in words]
print(clean)   # ['hello', 'world', 'python']

# Dict comprehension — same idea for dicts
names = ["alice", "bob", "charlie"]
name_lengths = {name: len(name) for name in names}
print(name_lengths)   # {'alice': 5, 'bob': 3, 'charlie': 7}

# Set comprehension — for unique results
sentence = "the cat sat on the mat"
unique_words = {word for word in sentence.split()}
print(unique_words)
```

```bash
python3 comprehensions.py
```

> **Readability rule:** If the comprehension spans more than one logical condition or transformation, use a regular `for` loop instead. Comprehensions are powerful, but clarity wins over cleverness.

---

## Step 5 — Building a Contact Book

Put all four collection types to work in a small program that manages contacts.

```python
# contact_book.py

contacts = {}   # dict of dicts: {username: {name, email, tags}}


def add_contact(username, name, email, tags=None):
    contacts[username] = {
        "name": name,
        "email": email,
        "tags": set(tags or []),
    }


def find_by_tag(tag):
    return [c for c in contacts.values() if tag in c["tags"]]


def all_tags():
    return {tag for c in contacts.values() for tag in c["tags"]}


def display(contact):
    tags_str = ", ".join(sorted(contact["tags"])) or "none"
    print(f"  {contact['name']} <{contact['email']}> [{tags_str}]")


# Populate
add_contact("alice", "Alice Smith",   "alice@example.com", ["python", "backend"])
add_contact("bob",   "Bob Jones",     "bob@example.com",   ["javascript", "frontend"])
add_contact("cara",  "Cara Williams", "cara@example.com",  ["python", "data"])

print("All contacts:")
for contact in contacts.values():
    display(contact)

print("\nPython developers:")
for contact in find_by_tag("python"):
    display(contact)

print(f"\nAll unique tags: {sorted(all_tags())}")
```

```bash
python3 contact_book.py
```

Expected output:
```
All contacts:
  Alice Smith <alice@example.com> [backend, python]
  Bob Jones <bob@example.com> [frontend, javascript]
  Cara Williams <cara@example.com> [data, python]

Python developers:
  Alice Smith <alice@example.com> [backend, python]
  Cara Williams <cara@example.com> [data, python]

All unique tags: ['backend', 'data', 'frontend', 'javascript', 'python']
```

---

## Step 6 — Sorting and Transforming Collections

Sorting structured data is a constant in real programs. Python's `sorted()` and the `key=` parameter give you powerful control without writing comparison functions from scratch.

```python
# Add to contact_book.py

# Sort contacts alphabetically by name
sorted_contacts = sorted(contacts.values(), key=lambda c: c["name"])
print("\nSorted by name:")
for c in sorted_contacts:
    display(c)

# Sort by number of tags (most → fewest)
by_tag_count = sorted(contacts.values(), key=lambda c: len(c["tags"]), reverse=True)
print("\nSorted by tag count:")
for c in by_tag_count:
    display(c)

# Build a summary dict using a dict comprehension
summary = {
    username: {"name": info["name"], "tag_count": len(info["tags"])}
    for username, info in contacts.items()
}
print("\nSummary:", summary)
```

```bash
python3 contact_book.py
```

> **`lambda`** is a way to write a small, anonymous function inline: `lambda c: c["name"]` is equivalent to `def get_name(c): return c["name"]`. Use lambdas only for simple one-expression key functions — anything more complex deserves a named function.

You now know Python's four core collection types and how to transform them efficiently. The next lab moves to reading and writing files — including CSV and JSON — which is how your programs will store and load data.
