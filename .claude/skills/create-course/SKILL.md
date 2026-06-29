---
name: create-course
description: Scaffold and author a new hands-on training course matching this repo's conventions (manifest.json, modules/labs, lab.md, checks.json). Use when the user asks to "create a course", "add a course", "write a new lab/module", or extend the courses catalog.
---

# Create a Course

Author a new course under `courses/<slug>/` that matches the exact structure, formatting, and conventions of the existing courses in this repository. Every course is a self-contained, hands-on, lab-based learning module that a learner completes in a VS Code workspace, with automated `checks.json` verifying their work.

## Before you start

Ask the user for (or infer from their request) the **topic**, **target audience** (default: new graduates), and any **scope constraints**. If a detail is unstated, follow the defaults below — do not block on questions that have a sensible default.

Then read **one existing course end-to-end** as your reference (a good default is `courses/python-crash-course/` for code-heavy topics, or `courses/agile-scrum-for-new-grads/` for conceptual/non-code topics). Match its tone, density, and structure. When in doubt, copy the shape of an existing course rather than inventing.

## Conventions (must match exactly)

### Directory layout
```
courses/<slug>/
├── .vscode/
│   └── settings.json
├── manifest.json
└── modules/
    ├── module-1/
    │   └── lab-1-1/
    │       ├── lab.md
    │       └── checks.json
    ├── module-2/
    │   ├── lab-2-1/{lab.md, checks.json}
    │   └── lab-2-2/{lab.md, checks.json}
    └── module-3/
        ├── lab-3-1/{lab.md, checks.json}
        └── lab-3-2/{lab.md, checks.json}
```

**Defaults (every existing course follows this):** 3 modules, 5 labs total distributed **1 + 2 + 2**, each lab `estimatedMinutes` 45–55. Only deviate if the user explicitly asks for a different size.

- `<slug>` is kebab-case, descriptive, often suffixed with the audience (e.g. `docker-for-new-grads`, `mermaid-diagramming`).
- `lab-N-M` = `lab-<moduleNumber>-<labNumberWithinModule>`.

### `.vscode/settings.json`
Identical in every course — copy verbatim:
```json
{
  "labs.courseSourceUrl": "${workspaceFolder}"
}
```

### `manifest.json`
See `templates/manifest.json` for the skeleton. Rules:
- `schemaVersion`: `1`
- `courseId`: `<slug>-2026`
- `title`: human-readable, **must match** the `# H1` of every lab's manifest entry and the catalog title
- `revision`: `YYYY-MM-DD-001` (use today's date)
- `publishedAt`: `YYYY-MM-DDT00:00:00Z` (same date)
- `modules[]`: `id` = `module-N`; `title` = `Module N — <Module Title>` (note the em-dash `—`); `order` = N (1-based)
- `labs[]`: `id` = `lab-N-M`; `title` = the lab's topic (matches the lab.md H1); `estimatedMinutes` 45–55; `path` = `modules/module-N/lab-N-M/lab.md`; `contentHash` = a short stable id: a 3-letter topic prefix + zero-padded sequence across the whole course, e.g. `dkr001`…`dkr005`, `py001`…`py005`.

### `lab.md`
See `templates/lab.md` for the skeleton. Formatting rules drawn from existing labs:
- Start with `# <Lab Title>` (matches manifest `title`).
- One intro paragraph beginning "In this lab you will …" describing the concrete outcome ("By the end you will have …").
- A `**Prerequisites:**` line.
- `---` horizontal rule between every step.
- Steps as `## Step N — <Step Title>` (em-dash, 1-based numbering).
- Use fenced code blocks with language tags (` ```bash `, ` ```python `, ` ```json `, etc.). Show the command to run AND, where useful, an **Expected output** block (plain ``` fence).
- Use `> **Term:** …` blockquote callouts to explain key concepts/idioms inline.
- Use Markdown tables for reference material (type tables, comparison tables).
- Each lab teaches by having the learner **create real files / run real commands** in their workspace — these are what `checks.json` verifies.
- End with a short closing paragraph summarizing what they learned and bridging to the next lab.

### `checks.json`
See `templates/checks.json` for the skeleton. Rules:
- `version`: `1`
- `steps[]`: each has a `stepIndex` and a `checks[]` array.
- **`stepIndex` is 0-based**: "## Step 1" → `stepIndex: 0`, "## Step 5" → `stepIndex: 4`.
- **Only include steps that produce a verifiable artifact** (a created file, file content, or a command result). Conceptual/reading steps get no entry. Existing labs typically have checks for 3–4 of their steps.
- Four check types only:
  | type | fields | verifies |
  |------|--------|----------|
  | `file_exists` | `path` | a file or directory exists |
  | `file_contains` | `path`, `contains` (literal substring) | file content |
  | `command_exits_zero` | `command` | command runs successfully (exit 0) |
  | `command_output_matches` | `command`, `pattern` (regex) | command stdout matches |
  - Every check also has a human-readable `description`.
  - Paths are relative to the workspace root and must match exactly what Step N's instructions told the learner to create.

### Catalog registration
After creating the course, add it to `courses/catalog.json` in the `courses[]` array, keeping the array **alphabetically sorted by `path`**:
```json
{ "title": "<Course Title>", "path": "<slug>" }
```

## Procedure

1. Read a reference course (full lab.md + checks.json + manifest.json) to internalize tone and density.
2. Decide the slug, the 3 module titles, and the 5 lab titles. Sketch the learning arc (foundations → practice → real-world).
3. Create the directory tree and `.vscode/settings.json`.
4. Write `manifest.json`.
5. Write each `lab.md` — substantive, hands-on, with runnable steps. This is the bulk of the work; do not stub it.
6. Write each `checks.json`, with `stepIndex` correctly 0-based and only for verifiable steps. Cross-check every `path`/`contains` against the actual instructions in the corresponding lab.
7. Register the course in `courses/catalog.json` (alphabetical by path).
8. Validate: run the check below and confirm all JSON parses and every manifest `path` resolves to a real file.

```bash
# from repo root — validate JSON and that all manifest lab paths exist
for f in courses/<slug>/manifest.json courses/<slug>/.vscode/settings.json courses/<slug>/modules/**/*/checks.json courses/catalog.json; do python3 -m json.tool "$f" >/dev/null && echo "OK $f" || echo "BAD $f"; done
python3 -c "import json,os; d=json.load(open('courses/<slug>/manifest.json')); [print('MISSING', l['path']) for m in d['modules'] for l in m['labs'] if not os.path.exists(os.path.join('courses/<slug>', l['path']))] or print('all lab paths exist')"
```

## Quality bar
- Labs must be genuinely educational and self-contained — a new grad should be able to follow them with only the stated prerequisites.
- Code must be correct and runnable. Verify any commands/snippets you can.
- Match the voice of existing courses: clear, encouraging, British/neutral spelling as seen in the repo (e.g. "authorised", "personalised"), concept callouts, real-world parallels.
- Every `checks.json` path/substring must actually be produced by following the lab — test your mental model against the lab text.
