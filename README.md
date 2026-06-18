# Braintrust — Claude Code plugin

**Build from what users actually said — not what your team assumes they want.**

Braintrust turns your scattered user research into an evidence-graded knowledge base your coding agent can consult, then lets you pressure-test any feature idea against it. It grades every piece of research by how trustworthy it is as evidence — separating *"last Tuesday I spent two hours fixing the CSV by hand"* (real, usable) from *"I'd definitely pay for that"* (a hypothetical you shouldn't build on).

A single Claude Code **plugin** (`braintrust`), distributed through the official community marketplace. **Zero backend** — everything runs in your Claude Code session and writes to a local folder; no server, no API keys, no data leaves your machine.

---

## Install

```bash
/plugin marketplace add anthropics/claude-plugins-community
/plugin install braintrust@claude-community
```

---

## The three skills

Braintrust is a pipeline: **figure out what you have → grade it → query it.**

### 1. `/braintrust:braintrust-setup` — *"what could I even use?"*

A 5-minute interview about where your real user knowledge lives — interviews, support tickets, sales calls, surveys, reviews, research docs. It produces a **ranked shortlist** of sources to feed in, ordered by how much firsthand user signal each is likely to hold, and flags the ones that are mostly internal opinion. Start here if you're not sure what counts as evidence or where to begin.

**Output:** a prioritized source list + your single highest-leverage starting point.

### 2. `/braintrust:braintrust-ingest <folder | file | URL | resource>` — *grade it and remember it*

Point it at a folder of transcripts, a single file, a URL, or a connected resource (a Linear issue, Confluence page, etc.). Braintrust:
- **copies** the source into a local `braintrust/` store (gitignored — your raw research never enters git),
- splits it into individual claims and **grades each one** `usable` / `borderline` / `discount` using a calibrated *Mom Test* evidence lens,
- and folds the results into a **running analysis** that accumulates across every ingest.

Run it as many times as you like — each run only grades what's new (deduped by content), and the synthesized analysis updates to reflect everything you've gathered.

**Output:** a count of new claims by verdict, the highest-leverage new theme, and an updated evidence base.

### 3. `/braintrust:braintrust-ask "<a feature, PR, PRD, or idea>"` — *does the evidence support this?*

Give it something you're thinking of building — a feature description, a pasted PRD, a PR, a Linear task, a design note. Braintrust reads your running analysis and judges it honestly:
- breaks the idea into **testable demand hypotheses** (the pain · the proposed solution · willingness to pay are evidenced separately),
- returns a **verdict per hypothesis**, weighted so real usable evidence outweighs opinion,
- backs each call with **specific verbatim quotes + where they came from**, and
- tells you **what evidence is missing** — the gaps you'd need to fill before building with confidence.

**Output:** a per-hypothesis verdict, cited quotes, an overall build-readiness call, and a required "what's missing" section.

---

## A walkthrough

You're a PM weighing whether to build a bulk-CSV importer. Instead of guessing:

```bash
# 1. Figure out what research you actually have
/braintrust:braintrust-setup
#    → "Your strongest source is the 12 customer interviews in Gong.
#       Start there. Your strategy decks will mostly grade as opinion."

# 2. Grade your strongest source and build the base
/braintrust:braintrust-ingest ~/research/gong-interviews
#    → "Graded 38 claims: 14 usable, 9 borderline, 15 discount.
#       Strongest new theme: manual data-prep is a recurring time sink."

# ...ingest more whenever you have it — tickets, a survey export, a call recording
/braintrust:braintrust-ingest ~/exports/zendesk-q1.csv

# 3. Pressure-test the actual idea
/braintrust:braintrust-ask "should we build a bulk CSV importer?"
```

…and you get back something like:

> **The *pain* is well-evidenced** — 4 usable claims describe manual CSV cleanup as a recurring burden ("Last Tuesday I spent two hours fixing the CSV…").
> **The specific *bulk-import* solution is only borderline** — one secondhand note that customers route around the export by hand.
> **Willingness to pay: no usable evidence** — only one "I'd pay for that" (a future promise — discounted).
> **Missing:** firsthand evidence on the solution shape and on pricing. Suggested next pull: 3 sales-call notes that touch pricing.

That's the point — it won't rubber-stamp an idea. It tells you *which parts* of your plan rest on real user behavior and which rest on wishful thinking.

---

## What it builds (project-local, gitignored)

```
your-repo/
  braintrust/
    .gitignore        # ignores both subdirs — verbatim user quotes never enter git
    sources/          # copies of what you ingested
    analysis/
      ledger.jsonl    # every graded claim: quote, verdict, provenance
      manifest.json   # what's been ingested (dedup + counts)
      digest.md       # synthesized analysis: demand signals by evidence strength + gaps
```

The evidence base stays **local to the project**. `ask` reads `digest.md` for the overview and pulls exact quotes from the ledger on demand.

---

## How the grading works

Every claim is scored on two axes from *The Mom Test* (Rob Fitzpatrick):

- **Who's speaking** — the user's own words and observed behavior (strongest) → secondhand summaries → internal opinion (not user data at all).
- **Fact or fluff** — a specific past event ("last Tuesday I…") → a real recurring workflow → a hypothetical ("I'd use it daily") → bare praise.

…plus contamination flags (leading questions, compliments, pitch-contaminated reactions). The result gates into:

| Verdict | Meaning |
|---|---|
| **usable** | Firsthand, specific, real behavior — trust it. |
| **borderline** | Real but relayed/generalized, or only thinly behavioral — corroborate before relying. |
| **discount** | Opinion, hypothetical, or compliment — don't build on it alone. |

The lens is **calibrated**: ~94% agreement with careful human judgment, and **zero hard usable↔discount errors** on a public held-out set — it flags the genuinely marginal middle as *borderline* rather than guessing. The single most valuable thing it surfaces is **a confident idea resting on weak evidence**.

---

## Current limitations of this plugin

- **Only as fresh as your last ingest** — there's no background sync. `ask` tells you when the base was last updated.
- **Grading runs in your session** — the rubric is calibrated, but the actual grades are produced by whatever model you're running; weaker models give noisier grades.
- **Local-only by design** — `sources/` and `analysis/` are gitignored because they contain verbatim user quotes, so the evidence base isn't shared with teammates or CI.
- **Absence of evidence ≠ evidence of absence** — a "no-evidence" verdict means your base is silent on something, not that users don't want it.

---

## Development

Bare plugin (manifest at `.claude-plugin/plugin.json`, skills under `skills/`, shared rubric in `reference/`), referenceable by any marketplace via `source: {github, repo: "tth4vb/braintrust-plugin"}`.

- Validate: `claude plugin validate .`
- Local install test: a gitignored `dev-marketplace/` is provided — `/plugin marketplace add ./dev-marketplace` then `/plugin install braintrust@braintrust-dev`.

## License

MIT
