# Creating an Application from a Prompt

In this lab you will use Claude to build a small but real application — a command-line task tracker — entirely through prompting. Along the way you will learn the anatomy of an effective prompt, why writing a short specification *before* you prompt produces dramatically better results, and how to iterate with follow-up prompts instead of starting over. By the end you will have a working application and, just as importantly, a reusable prompt library and a prompt journal that records what worked and why.

**Prerequisites:** Access to Claude (Claude Code in your terminal, the desktop app, or claude.ai). Python 3.10 or later installed (`python3 --version`). A terminal and VS Code. No prior prompt-engineering experience required.

---

## Step 1 — Set Up Your Workspace and Prompt Journal

Good prompt engineering is a discipline, not a lucky guess. Throughout this course you will keep two things: a **prompt library** (reusable prompts you can adapt later) and a **prompt journal** (a running log of what you asked, what you got back, and what you changed). Treat them like a lab notebook.

Create your workspace:

```bash
mkdir claude-prompting
cd claude-prompting
mkdir prompts
```

Create the journal file:

```bash
touch prompt-journal.md
```

Open `prompt-journal.md` and add a heading so you have somewhere to write as you go:

```markdown
# Prompt Journal

A running log of prompts I tried, the results, and what I learned.

## Lab 1 — Creating an Application
```

> **Why a journal?** The single biggest skill in prompt engineering is noticing *which change to a prompt caused which change in the output*. If you only ever look at the final answer, you learn nothing. Writing down the before/after turns every interaction into a lesson.

---

## Step 2 — The Anatomy of an Effective Prompt

A vague prompt produces a vague application. Almost every strong prompt contains four ingredients. Memorise them — you will use them in every module of this course.

| Ingredient | What it answers | Example |
|------------|-----------------|---------|
| **Context** | What is the situation? What already exists? | "I am building a command-line tool in Python 3. There is no existing code yet." |
| **Task** | What exactly do you want done? | "Create a task tracker that adds, lists, completes, and deletes tasks." |
| **Constraints** | What rules must the output obey? | "Use only the standard library. Store data in a JSON file. One file named `app.py`." |
| **Format** | What shape should the answer take? | "Give me the complete file, then a short note on how to run it." |

Capture these four principles in your prompt library so you can refer back to them:

```bash
touch prompts/principles.md
```

Add this content to `prompts/principles.md`:

```markdown
# Prompting Principles — C.T.C.F.

Every effective prompt should include:

1. **Context** — the situation and what already exists.
2. **Task** — the single, specific thing I want done.
3. **Constraints** — the rules the output must obey (language, libraries, file names).
4. **Format** — the shape of the response I want back.

Rule of thumb: if I would struggle to hand this prompt to a brand-new
colleague and expect the right result, Claude will struggle too.
```

> **Specific beats clever.** You do not need magic words or elaborate "you are an expert" preambles. Clear context, a concrete task, explicit constraints, and a stated format outperform any incantation.

---

## Step 3 — Write a Specification Before You Prompt

The most common beginner mistake is prompting "build me a task app" and then being disappointed. Spend three minutes writing a small spec first. The spec *becomes* the context and constraints of your prompt.

Create the application folder and a spec:

```bash
mkdir task-tracker
touch task-tracker/spec.md
```

Fill in `task-tracker/spec.md`:

```markdown
# Task Tracker — Specification

## Goal
A single-file command-line app to track personal tasks.

## Requirements
- Add a task with a title.
- List all tasks, showing an id, status, and title.
- Mark a task complete by id.
- Delete a task by id.

## Constraints
- Python 3, standard library only (no pip installs).
- Persist tasks to `tasks.json` in the current directory.
- Single file named `app.py`.
- Usage: `python3 app.py <command> [args]`.

## Out of scope
- Due dates, priorities, multiple users. (Maybe later.)
```

> **The spec is reusable leverage.** Notice that the spec already contains three of your four prompt ingredients — context (goal), task (requirements), and constraints. Writing it once means you never have to hold all of it in your head while you prompt.

---

## Step 4 — Craft the Scaffolding Prompt

Now turn the spec into a real prompt and save it to your library. Saving prompts as files lets you reuse and refine them rather than retyping.

```bash
touch prompts/01-scaffold.md
```

Write your first build prompt into `prompts/01-scaffold.md`:

```markdown
# Prompt 01 — Scaffold the task-tracker app

I am building a command-line tool in Python 3. There is no existing code.

Task: create a task tracker as described by the specification below.

Specification:
- Add a task with a title.
- List all tasks showing id, status (todo/done), and title.
- Mark a task complete by id.
- Delete a task by id.

Constraints:
- Python 3, standard library only.
- Persist tasks to tasks.json in the current directory.
- A single file named app.py.
- Command-line usage: python3 app.py <command> [args]
  (commands: add, list, done, delete)

Format:
- Give me the complete contents of app.py.
- Then give me four example commands I can run to test it.
```

> **Name files in your prompt.** Because the prompt explicitly says "a single file named `app.py`", Claude will produce exactly that file name. Telling the model precisely what to call things is how you keep generated output predictable enough to build on.

---

## Step 5 — Generate the Application with Claude

Now send your scaffolding prompt to Claude. If you are using **Claude Code**, run it from inside the `claude-prompting` directory so Claude can write the file directly:

```bash
claude
```

Then paste the contents of `prompts/01-scaffold.md`, adding one line at the top so Claude writes the file for you:

```
Create the file at task-tracker/app.py. <then paste your prompt 01>
```

If you are using the desktop or web app, copy the generated `app.py` contents into `task-tracker/app.py` yourself.

Either way, you should end up with the file in place:

```bash
ls task-tracker
```

Expected output:
```
app.py  spec.md
```

Run the app to confirm it works:

```bash
cd task-tracker
python3 app.py add "Write my prompt journal"
python3 app.py list
```

Expected output (yours will look similar):
```
Added task 1: Write my prompt journal
[1] todo  Write my prompt journal
```

> **Read before you run — every time.** Generated code is a draft, not gospel. Skim `app.py`: does it only use the standard library? Does it write to `tasks.json`? Treating output as a pull request to review is the habit that separates productive use of Claude from blind copy-paste.

---

## Step 6 — Iterate with Follow-Up Prompts

The first version is rarely the final version, and that is fine — iteration is the point. Instead of rewriting your whole prompt, give Claude a small, targeted follow-up that references what already exists.

Save your refinement prompt:

```bash
cd ..
touch prompts/02-refine.md
```

Write a focused change request into `prompts/02-refine.md`:

```markdown
# Prompt 02 — Refine the task-tracker app

The app.py we just built works. Make two small, focused changes:

1. When listing tasks, show completed tasks with a check mark (e.g. "[x]")
   and todo tasks with a space (e.g. "[ ]").
2. If the user runs an unknown command, print a short usage message
   listing the valid commands and exit with a non-zero status.

Keep everything else the same. Show me only the parts of app.py that change.
```

Send that follow-up to Claude and apply the result. Then record the iteration in your journal — this is the most valuable habit in the whole lab:

Append to `prompt-journal.md`:

```markdown
### Iteration 1 — display and error handling
- Before: list showed plain "todo/done"; unknown commands crashed.
- Change asked for: check marks in list, friendly usage message.
- Result: <did it work first try? what did you have to clarify?>
- Lesson: small scoped follow-ups beat re-describing the whole app.
```

> **Scope your follow-ups.** "Make two small, focused changes" with "keep everything else the same" tells Claude not to rewrite working code. Unbounded requests like "make it better" invite sprawling rewrites you then have to re-review.

---

## Step 7 — Review and Verify

Finish by exercising the whole app the way a user would, confirming each requirement from your spec is met:

```bash
cd task-tracker
python3 app.py add "Finish lab 1"
python3 app.py done 1
python3 app.py list
python3 app.py delete 1
python3 app.py list
```

Walk down your `spec.md` requirements one by one and tick them off against the behaviour you just observed. If something is missing, that is simply your next follow-up prompt.

You have now built a real application through prompting, and — more importantly — you have a repeatable method: write a spec, turn it into a C.T.C.F. prompt, generate, review, and iterate in small scoped steps. You also have a prompt library and journal you will keep extending. In the next module you will point these same skills at a problem every engineer faces constantly: making sense of a codebase you did not write.
