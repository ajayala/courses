# Investigating Bugs and Tracing Behaviour

In this lab you will use Claude to diagnose a real bug — not by asking "fix this", but by guiding it through a proper investigation: reproduce, hypothesise, locate the root cause, and only then fix. You will work with a small Python program that contains a classic, genuinely tricky bug. By the end you will have a written root-cause analysis and a verified fix, plus a debugging prompt pattern you can reuse on any codebase.

**Prerequisites:** You completed Lab 1 and have the `claude-prompting` workspace. Python 3.10+ and access to Claude. A terminal and VS Code.

---

## Step 1 — Create the Buggy Program

So that everyone investigates the same defect, you will start from a fixed, deliberately buggy program. Create a folder and the file exactly as shown.

```bash
cd claude-prompting
mkdir debugging
```

Create `debugging/cart.py` with this exact content:

```python
# cart.py — a tiny shopping cart with a hidden bug
def add_item(name, price, cart=[]):
    """Add an item to a cart and return the cart."""
    cart.append({"name": name, "price": price})
    return cart


def total(cart):
    return sum(item["price"] for item in cart)


if __name__ == "__main__":
    # Alice shops
    alice = add_item("book", 12.0)
    add_item("pen", 3.0, alice)
    print("Alice's cart:", alice)
    print("Alice's total:", total(alice))

    # Bob starts a brand-new cart
    bob = add_item("coffee", 4.5)
    print("Bob's cart:", bob)
    print("Bob's total:", total(bob))
```

Run it and observe the output carefully:

```bash
cd debugging
python3 cart.py
```

Expected (buggy) output:
```
Alice's cart: [{'name': 'book', 'price': 12.0}, {'name': 'pen', 'price': 3.0}]
Alice's total: 15.0
Bob's cart: [{'name': 'book', 'price': 12.0}, {'name': 'pen', 'price': 3.0}, {'name': 'coffee', 'price': 4.5}]
Bob's total: 19.5
```

Bob never added a book or a pen, yet they are in his cart. Something is leaking state between calls.

> **Reproduce before you investigate.** The first move in any bug hunt is a reliable reproduction. You now have one: run `cart.py` and Bob inherits Alice's items every time. A bug you can reproduce on demand is a bug you can fix with confidence.

---

## Step 2 — Write a Reproduction Report

Before prompting, write down exactly what you observed. This becomes the context for your debugging prompt and forces you to separate *symptom* from *guess*.

```bash
cd ..
touch debugging/bug-report.md
```

Start `debugging/bug-report.md`:

```markdown
# Bug Report — shopping cart

## Symptom
A newly created cart is not empty: it already contains items added to a
previous cart.

## Steps to reproduce
1. Run `python3 cart.py`.
2. Observe Bob's cart contains Alice's "book" and "pen".

## Expected vs actual
- Expected: Bob's cart has only "coffee"; total 4.5.
- Actual: Bob's cart has book, pen, coffee; total 19.5.

## Root cause
(to be filled in)

## Fix
(to be filled in)
```

> **Symptom is not cause.** "Bob's total is wrong" is a symptom. The cause is something specific in the code. Keeping these in separate sections of your report stops you from "fixing" the number while leaving the real defect in place.

---

## Step 3 — Prompt Claude to Find the Root Cause

Now bring Claude in as an investigator. Notice what this prompt does *not* do: it does not ask for a fix yet. Asking for the diagnosis first keeps you in control and teaches you something.

Save the prompt to your library:

```bash
touch prompts/06-debug.md
```

Write your debugging prompt into `prompts/06-debug.md`:

```markdown
# Prompt 06 — Diagnose a bug (root cause first)

Here is a program and a reproduction.

<paste the full contents of cart.py here>

Symptom: a brand-new cart created by add_item already contains items from a
previous call. Reproduction: running the file shows Bob's cart containing
Alice's items.

Do NOT fix it yet. First:
1. Explain the root cause precisely, in terms of how Python evaluates this code.
2. Point to the exact line responsible.
3. Explain why it produces this specific symptom (state shared across calls).

Then, and only then, propose the smallest correct fix and show the changed lines.
```

Send it to Claude. The root cause is Python's **mutable default argument**: `cart=[]` is evaluated once when the function is defined, so every call that omits `cart` shares the same list object.

> **"Diagnose, don't fix (yet)."** Leading with the root cause does two things: it teaches you the underlying concept (so you recognise it next time), and it lets you sanity-check the diagnosis before any code changes. A fix you do not understand is a fix you cannot trust.

---

## Step 4 — Apply and Verify the Fix

Record what you learned, then apply the canonical fix for this bug: default to `None` and create a fresh list inside the function.

Update `debugging/cart.py` so the function reads:

```python
def add_item(name, price, cart=None):
    """Add an item to a cart and return the cart."""
    if cart is None:
        cart = []
    cart.append({"name": name, "price": price})
    return cart
```

Run it again to confirm the bug is gone:

```bash
cd debugging
python3 cart.py
```

Expected (fixed) output:
```
Alice's cart: [{'name': 'book', 'price': 12.0}, {'name': 'pen', 'price': 3.0}]
Alice's total: 15.0
Bob's cart: [{'name': 'coffee', 'price': 4.5}]
Bob's total: 4.5
```

Now complete the **Root cause** and **Fix** sections of `debugging/bug-report.md` in your own words. Make sure the Root cause section names the real cause:

```markdown
## Root cause
The default argument `cart=[]` is evaluated once, at function-definition time.
Every call that omits `cart` reuses that same shared list, so items leak
between calls (a "mutable default argument" bug).

## Fix
Default `cart=None` and create a new list inside the function when none is
passed. Verified: Bob's cart now contains only coffee, total 4.5.
```

> **Verify against the original reproduction.** The fix is only proven when the exact steps that produced the bug now produce the correct result. Re-run the same reproduction — do not invent a new, easier test that happens to pass.

---

## Step 5 — Generalise the Pattern

A single fixed bug is worth more if you extract the reusable lesson. Capture the debugging pattern so you can apply it to bugs that are nothing like this one.

Append to your `prompt-journal.md`:

```markdown
## Lab 2 — Debugging
### Reusable debugging pattern
1. Reproduce reliably and write down symptom vs expected.
2. Give Claude the code + reproduction; ask for ROOT CAUSE first, no fix.
3. Confirm the cause makes sense, then ask for the smallest fix.
4. Re-run the original reproduction to verify.
Lesson learned: mutable default arguments share one object across calls.
```

> **Patterns outlive bugs.** You will probably never see this exact cart code again, but you will see shared-state bugs, off-by-one errors, and silent type coercions for the rest of your career. The four-step pattern — reproduce, diagnose, fix small, re-verify — applies to all of them.

You can now drive Claude through a disciplined bug investigation and produce a written root-cause analysis rather than a mystery patch. In the final module you will turn from understanding existing code to researching something new — and then apply everything to a substantial real-world task.
