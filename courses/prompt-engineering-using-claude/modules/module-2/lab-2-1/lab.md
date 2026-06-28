# Exploring an Unfamiliar Codebase

In this lab you will use Claude to get oriented in a codebase you have never seen — one of the most common situations you will face as a new engineer. You will learn the "zoom out, then zoom in" investigation pattern: start with a high-level map, then drill into the parts that matter. By the end you will have produced a codebase map and a glossary that turn an intimidating wall of files into something you can reason about.

**Prerequisites:** You completed Lab 1 and have the `claude-prompting` workspace. Access to Claude (Claude Code recommended, as it can read files directly). Git installed (`git --version`).

---

## Step 1 — Choose a Subject and Set Up

You need a codebase to investigate. The best subject is real and not yours. Clone a small, well-known open-source project — or reuse the `task-tracker` you built in Lab 1 if you would rather work fully offline.

Move into your workspace and create a folder for your investigation notes:

```bash
cd claude-prompting
mkdir investigation
```

Clone a small project to explore (any small repo works; this one is tiny and dependency-light):

```bash
git clone https://github.com/pallets/click.git investigation/subject
```

> **Why investigate someone else's code?** When you wrote the code, your understanding is in your head and prompting feels pointless. The real skill — and the real lab — is building understanding of code you have *no* context for. That is what most of your career looks like.

---

## Step 2 — Zoom Out: Map the Territory

Resist the urge to open a random file and start reading line by line. First ask Claude for the big picture. Save the prompt to your library, then record what you learn.

```bash
touch prompts/03-map.md
```

Write your orientation prompt into `prompts/03-map.md`:

```markdown
# Prompt 03 — Map an unfamiliar codebase

Context: I have just been given this repository and have no prior knowledge
of it. Treat me as a new joiner.

Task: give me a high-level orientation. Specifically:
1. What does this project do, in two sentences?
2. What are the main top-level directories and what is each responsible for?
3. Where is the entry point / public API a user would start from?
4. What are the 3–5 files I should read first to understand the core?

Constraints: base your answer on the actual files in this repo, not general
knowledge. If you are unsure about something, say so.

Format: a short summary, then a table of "file or directory | responsibility".
```

If you are using Claude Code, run `claude` from inside `investigation/subject` so it can read the real files. Capture the response in a map file:

```bash
touch investigation/codebase-map.md
```

Record Claude's overview in `investigation/codebase-map.md`, using these headings so your notes stay structured:

```markdown
# Codebase Map — subject

## What it does

## Top-level layout
| Path | Responsibility |
|------|----------------|

## Where to start reading
```

> **Insist on "based on the actual files".** Models can pattern-match to how a *typical* project of this kind is laid out and quietly invent structure. The constraint "base your answer on the actual files in this repo… if you are unsure, say so" pushes Claude to ground its answer in what is really there.

---

## Step 3 — Zoom In: Explain a Specific File

Now that you know where to start, drill into one of the files Claude pointed you to. The goal of a drill-down prompt is a guided reading, not a wall of paraphrase.

Pick one core file from your map (for the `click` repo, `src/click/core.py` is a good choice) and ask:

```markdown
# Prompt 04 — Explain a specific file (paste into Claude)

Walk me through the file <path-to-file>. I want to understand it, not just
read a summary.

1. What is this file's single main responsibility?
2. What are the key classes/functions, and how do they relate?
3. Point to the 2–3 most important lines or sections and explain why they matter.
4. What would I be likely to change in here, and what should I be careful of?

If something in the file is genuinely confusing or looks like a workaround,
say so rather than smoothing it over.
```

> **Ask for the "why", not just the "what".** Anyone can read code top to bottom. The value Claude adds is connecting the pieces — which function calls which, why a class exists, what a non-obvious line is guarding against. Prompts that ask for relationships and rationale beat "summarise this file".

---

## Step 4 — Build a Glossary of Domain Terms

Every codebase has its own vocabulary — names that mean something specific in this project. Capturing them is the fastest way to stop feeling lost. Create a glossary and have Claude help you populate it.

```bash
touch investigation/glossary.md
```

Ask Claude:

```markdown
# Prompt 05 — Extract the project's vocabulary (paste into Claude)

From this codebase, list the 8–12 most important domain-specific terms,
type names, or concepts a new contributor must understand (for example
class names, key abstractions, or recurring nouns in the code).

For each: the term, a one-line plain-English definition, and where in the
code it is defined or most clearly used.
```

Record the result in `investigation/glossary.md` with a Terms section:

```markdown
# Glossary — subject

## Terms
| Term | Meaning | Where it lives |
|------|---------|----------------|
```

> **A glossary is a force multiplier.** Once you can name the core abstractions, every future prompt gets sharper because you can use the project's own words. "How does the `Context` object get passed to a `Command`?" is a far better question than "how does the data move around?".

---

## Step 5 — Verify One Claim Yourself

Never let your understanding rest entirely on an explanation — yours or Claude's. Pick one specific claim from your map or glossary and confirm it against the actual code with a quick search.

For example, if Claude said a certain class is defined in a certain file:

```bash
grep -rn "class Context" investigation/subject/src
```

Add a short "Verified" note to the bottom of your `codebase-map.md`:

```markdown
## Verified
- Claim: <the claim you checked>
- How I checked: <command or file I opened>
- Result: confirmed / corrected
```

> **Trust, then verify.** Claude is an outstanding guide to unfamiliar code, but it can occasionally point to the wrong line or describe code that was refactored away. A ten-second `grep` to confirm the one claim your work depends on is always worth it.

You now have a repeatable investigation workflow — zoom out to a map, zoom in to key files, build a glossary, and verify the load-bearing claims. In the next lab you will use the same investigative mindset for a sharper purpose: tracking down and explaining a bug.
