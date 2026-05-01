# Reading Files, Writing Files, and Working with JSON

In this lab you will learn how Python programs persist and exchange data. You will read and write plain text files, parse CSV data using the standard library, serialize and deserialize JSON, and handle the errors that arise when files are missing or malformed. These skills are the foundation of every data pipeline, config loader, and API client you will write.

**Prerequisites:** Lab 2-1 completed. `python-lab/contact_book.py` exists.

---

## Step 1 — Reading and Writing Text Files

Python's built-in `open()` function gives you a file object. Always use it as a context manager (`with` statement) — this guarantees the file is closed even if an exception occurs.

```python
# file_io.py  (create inside python-lab/)

# Writing a text file
with open("notes.txt", "w") as f:
    f.write("Line one\n")
    f.write("Line two\n")
    f.writelines(["Line three\n", "Line four\n"])

# Reading the whole file at once
with open("notes.txt", "r") as f:
    content = f.read()
    print(repr(content))   # shows \n characters

# Reading line by line — memory-efficient for large files
with open("notes.txt", "r") as f:
    for line in f:
        print(line.strip())   # strip() removes trailing \n

# Appending to an existing file
with open("notes.txt", "a") as f:
    f.write("Line five\n")

# Reading all lines into a list
with open("notes.txt", "r") as f:
    lines = f.readlines()   # ['Line one\n', 'Line two\n', ...]
print(f"File has {len(lines)} lines")
```

```bash
cd python-lab
python3 file_io.py
```

| Mode | Meaning |
|------|---------|
| `"r"` | Read (default). Raises `FileNotFoundError` if file missing. |
| `"w"` | Write. Creates the file; **truncates** if it already exists. |
| `"a"` | Append. Creates if missing; never truncates. |
| `"x"` | Exclusive create. Raises `FileExistsError` if file exists. |
| `"rb"` / `"wb"` | Binary read/write — for images, PDFs, etc. |

---

## Step 2 — Working with Paths

Hardcoding path strings like `"data/file.txt"` breaks when the working directory changes. Python's `pathlib` module gives you a clean, cross-platform way to build and inspect paths.

```python
# Add to file_io.py

from pathlib import Path

# Build paths
base = Path("python-lab-data")
base.mkdir(exist_ok=True)   # create directory if it doesn't exist

output_file = base / "report.txt"   # / operator joins path segments

# Write using Path directly
output_file.write_text("Report content\n")

# Read using Path directly
content = output_file.read_text()
print(content)

# Inspect paths
print(output_file.name)       # report.txt
print(output_file.stem)       # report
print(output_file.suffix)     # .txt
print(output_file.parent)     # python-lab-data
print(output_file.exists())   # True

# List all .txt files in a directory
for txt_file in base.glob("*.txt"):
    print(txt_file)
```

```bash
python3 file_io.py
```

> **Always use `pathlib.Path`** for new code. It replaces the older `os.path` module with a more readable, object-oriented API. On Windows, `Path` automatically uses backslashes; on Linux/Mac it uses forward slashes — your code works on both without changes.

---

## Step 3 — CSV Files

CSV (comma-separated values) is the universal format for tabular data. Python's standard library includes a `csv` module that handles quoting, escaping, and different delimiters automatically.

```python
# csv_demo.py

import csv
from pathlib import Path

# Write a CSV file
employees = [
    {"name": "Alice",   "department": "Engineering", "salary": 85000},
    {"name": "Bob",     "department": "Marketing",   "salary": 72000},
    {"name": "Charlie", "department": "Engineering", "salary": 91000},
]

output = Path("employees.csv")
with output.open("w", newline="") as f:   # newline="" is required on Windows
    writer = csv.DictWriter(f, fieldnames=["name", "department", "salary"])
    writer.writeheader()
    writer.writerows(employees)

print(output.read_text())

# Read the CSV back
with output.open("r", newline="") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(f"{row['name']:10} {row['department']:15} ${int(row['salary']):,}")
```

```bash
python3 csv_demo.py
```

Expected output:
```
name,department,salary
Alice,Engineering,85000
Bob,Marketing,72000
Charlie,Engineering,91000

Alice      Engineering     $85,000
Bob        Marketing       $72,000
Charlie    Engineering     $91,000
```

> **`DictReader` / `DictWriter`** are almost always preferable to the raw `reader` / `writer` because they use column names instead of positional indices — your code stays correct when columns are reordered.

---

## Step 4 — JSON

JSON is the standard format for configuration files and API responses. Python's `json` module converts between JSON text and Python dicts/lists with two functions: `json.dumps()` (to string) and `json.loads()` (from string), plus `json.dump()` / `json.load()` for files.

```python
# json_demo.py

import json
from pathlib import Path

config = {
    "app_name": "MyApp",
    "version": "1.0.0",
    "debug": False,
    "database": {
        "host": "localhost",
        "port": 5432,
        "name": "myapp_db",
    },
    "allowed_origins": ["http://localhost:3000", "https://myapp.com"],
}

# Serialize to a JSON file
config_path = Path("config.json")
with config_path.open("w") as f:
    json.dump(config, f, indent=2)   # indent=2 makes the file human-readable

print(config_path.read_text())

# Deserialize from the file
with config_path.open("r") as f:
    loaded = json.load(f)

print(loaded["database"]["port"])           # 5432
print(type(loaded["debug"]))                # <class 'bool'>  — JSON booleans become Python booleans

# Inline serialization (useful for logging and APIs)
subset = {"host": loaded["database"]["host"], "port": loaded["database"]["port"]}
print(json.dumps(subset))                   # {"host": "localhost", "port": 5432}
```

```bash
python3 json_demo.py
```

| Python type | JSON equivalent |
|-------------|----------------|
| `dict` | object `{}` |
| `list` / `tuple` | array `[]` |
| `str` | string `""` |
| `int` / `float` | number |
| `True` / `False` | `true` / `false` |
| `None` | `null` |

---

## Step 5 — Saving and Loading the Contact Book

Apply file I/O to persist the contact book from the previous lab. The program will load existing contacts from a JSON file at startup and save them back on exit.

```python
# contact_book_v2.py

import json
from pathlib import Path

DATA_FILE = Path("contacts.json")


def load_contacts():
    if not DATA_FILE.exists():
        return {}
    with DATA_FILE.open("r") as f:
        raw = json.load(f)
    # JSON doesn't have a set type, so tags are stored as lists
    return {username: {**data, "tags": set(data["tags"])} for username, data in raw.items()}


def save_contacts(contacts):
    serialisable = {
        username: {**data, "tags": sorted(data["tags"])}
        for username, data in contacts.items()
    }
    with DATA_FILE.open("w") as f:
        json.dump(serialisable, f, indent=2)


def add_contact(contacts, username, name, email, tags=None):
    contacts[username] = {"name": name, "email": email, "tags": set(tags or [])}


contacts = load_contacts()
add_contact(contacts, "diana", "Diana Prince", "diana@example.com", ["python", "security"])
add_contact(contacts, "evan",  "Evan Green",   "evan@example.com",  ["go", "backend"])
save_contacts(contacts)

print(f"Saved {len(contacts)} contacts to {DATA_FILE}")
print(DATA_FILE.read_text())
```

```bash
python3 contact_book_v2.py
```

Run it a second time — the contacts persist between runs.

---

## Step 6 — Handling Missing Files and Malformed Data

File operations fail in predictable ways. Catching specific exceptions keeps your program useful instead of crashing with an unhelpful traceback.

```python
# robust_reader.py

import json
from pathlib import Path


def read_json_file(path):
    """Load a JSON file, returning None with a clear message on any failure."""
    p = Path(path)

    if not p.exists():
        print(f"File not found: {path}")
        return None

    if p.suffix != ".json":
        print(f"Expected a .json file, got: {p.suffix}")
        return None

    try:
        with p.open("r") as f:
            return json.load(f)
    except json.JSONDecodeError as e:
        print(f"Invalid JSON in {path}: {e}")
        return None
    except PermissionError:
        print(f"Permission denied reading {path}")
        return None


# Test with the config we created earlier
data = read_json_file("config.json")
if data:
    print("Loaded config:", data["app_name"])

# Test with a missing file
read_json_file("does_not_exist.json")

# Test with malformed JSON
Path("bad.json").write_text("{not valid json")
read_json_file("bad.json")
```

```bash
python3 robust_reader.py
```

Expected output:
```
Loaded config: MyApp
File not found: does_not_exist.json
Invalid JSON in bad.json: ...
```

> **Catch specific exceptions**, not bare `except Exception`. Catching `json.JSONDecodeError` separately from `PermissionError` lets you give the user a useful message for each failure mode — and avoids accidentally swallowing unexpected bugs.

You can now read and write text files, parse CSV and JSON, build robust paths with `pathlib`, and handle file-related errors gracefully. The next module moves up a level: organising your code into classes, handling errors systematically, and packaging your work as a reusable module.
