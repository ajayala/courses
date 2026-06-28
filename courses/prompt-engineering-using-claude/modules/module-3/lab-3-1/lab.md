# Researching Technical Topics with Claude

In this lab you will use Claude as a research partner for an open technical question — the kind you face whenever you must choose a library, an approach, or an architecture. The skill is not "ask and accept"; it is structuring the question, demanding trade-offs over recommendations, and verifying the claims that matter. By the end you will have a structured research brief you could hand to a teammate, and a verification habit that protects you from confidently-wrong answers.

**Prerequisites:** You completed Labs 1 and 2 and have the `claude-prompting` workspace. Access to Claude. A terminal and VS Code.

---

## Step 1 — Frame the Research Question

Vague questions get vague research. A good research prompt starts from a sharp question with explicit decision criteria. We will use a concrete running question for this lab:

> *"For a small Python web service, should I use Flask or FastAPI?"*

Set up your research folder and capture the framing:

```bash
cd claude-prompting
mkdir research
touch research/brief.md
```

Start `research/brief.md` by writing the question and what "good" looks like:

```markdown
# Research Brief — Flask vs FastAPI for a small Python web service

## Question
For a small internal web service (a few endpoints, JSON in/out), should we
use Flask or FastAPI?

## Decision criteria
- Learning curve for a new-grad team.
- Built-in request validation and docs.
- Performance for modest traffic.
- Ecosystem and long-term maintenance.
```

> **Decisions need criteria.** "Which is better?" has no answer; "which better fits *these* criteria for *this* context?" does. Stating your criteria up front turns research from trivia-gathering into decision support — and it tells Claude exactly what to compare.

---

## Step 2 — Prompt for Trade-offs, Not a Verdict

A weak prompt asks "what's the best framework?" and gets an opinion. A strong prompt asks for a structured comparison against your criteria, with the downsides made explicit.

Save the prompt:

```bash
touch prompts/07-research.md
```

Write your research prompt into `prompts/07-research.md`:

```markdown
# Prompt 07 — Structured technical research

Context: a new-grad team is choosing a Python web framework for a small
internal JSON service with a few endpoints and modest traffic.

Task: compare Flask and FastAPI against these criteria:
learning curve, built-in validation and docs, performance, ecosystem/maintenance.

Constraints:
- Give the trade-offs, not just a winner. For each option include at least
  one real downside.
- Flag anything that depends on version or has changed recently, and tell me
  what I should verify myself.
- Do not invent statistics or benchmark numbers; if you are unsure, say so.

Format: a comparison table (criterion | Flask | FastAPI), then a short
"it depends on…" paragraph, then a tentative recommendation for our context.
```

> **Ask for the downsides on purpose.** Left to its own devices, a model tends to be agreeable and may oversell whichever option it names first. Explicitly requiring "at least one real downside" for each option surfaces the trade-offs you actually need to make a decision.

---

## Step 3 — Capture the Findings as a Comparison

Send the prompt to Claude and record the result in your brief. Structure matters: a table you can scan beats prose you have to re-read.

Add to `research/brief.md`:

```markdown
## Comparison
| Criterion | Flask | FastAPI |
|-----------|-------|---------|
| Learning curve |  |  |
| Validation & docs |  |  |
| Performance |  |  |
| Ecosystem & maintenance |  |  |

## It depends on…
(the conditions that would push the decision one way or the other)

## Tentative recommendation
(your call, for your context — and your confidence level)
```

> **You own the conclusion.** Claude can lay out the landscape, but the recommendation is yours to make and to justify. Writing the "tentative recommendation" in your own words — with a confidence level — is what turns research into a decision you can defend.

---

## Step 4 — Separate Fact from Claim and Verify

This is the step that separates real research from copy-paste. Some statements from any AI are stable facts; others are claims that need checking — especially version-specific behaviour, benchmarks, and "X supports Y" assertions.

Create a verification checklist in your brief:

```markdown
## Claims to verify
| Claim | How I'll check it | Verified? |
|-------|-------------------|-----------|
| FastAPI has built-in request validation | official FastAPI docs | |
| Flask needs an extension for the same | official Flask docs | |
```

Then actually verify at least one claim against a primary source. If you have Claude Code or web access, you can have Claude fetch and cite the official documentation — but confirm the source is the real project's docs, not a paraphrase:

```markdown
# Prompt 08 — Verify a specific claim (paste into Claude)

Verify this single claim against the official documentation and quote the
relevant line with a link: "FastAPI performs request body validation using
Python type hints and Pydantic." If the docs do not clearly support it,
say so.
```

> **Hallucination lives in the specifics.** Broad explanations are usually reliable; precise claims — a version number, a benchmark figure, a specific API name — are where models are most likely to be confidently wrong. Verify the specific claims your decision rests on, and cite a primary source.

---

## Step 5 — Write the Decision Summary

Close the loop with a short, honest summary that someone could act on without reading the whole brief.

Add a final section to `research/brief.md`:

```markdown
## Decision summary
- Recommendation: <option>, because <one or two reasons tied to our criteria>.
- Confidence: <low / medium / high>, because <what is verified vs assumed>.
- Open questions: <anything still unverified that could change the call>.
```

> **State your confidence.** "FastAPI, high confidence" and "FastAPI, low confidence — performance claim unverified" are very different messages to a teammate. Reporting confidence and open questions honestly is a hallmark of trustworthy technical work — and exactly the standard you should hold AI-assisted research to.

You now have a research method that produces a defensible, source-checked brief rather than an unverified opinion: frame the question with criteria, prompt for trade-offs, capture a structured comparison, verify the load-bearing claims, and own the conclusion. In the final lab you will put the entire course together on a substantial real-world task: researching and implementing CAS authentication in a Java web application.
