---
name: braintrust-setup
description: Interview the user about where their user-research and customer knowledge lives, then produce a customized ingestion plan — a prioritized list of sources to pull into a local folder so a coding agent can read real user evidence. Use when the user wants to "set up Brain Trust", "plan what to ingest", "figure out what knowledge to pull in", "onboard my research sources", or is starting to gather user context for an agent to use.
---

# Brain Trust Setup — source quiz → customized ingestion plan

Goal: in one short interview, figure out **where this user's real user-knowledge lives**, then hand them a concrete, prioritized **ingestion plan** — which sources to pull into a local `braintrust/sources/` folder, in what order, why, and how. The plan is shaped by their answers; do not produce a generic checklist.

Pair with the `evidence-lens` skill: setup decides *what to pull in*, the lens decides *what's trustworthy once it's in*. Bias the plan toward sources likely to contain **firsthand, specific user behavior** (the lens's "usable" tier) over opinion/secondhand sources.

## How to run it

Conduct the quiz conversationally — **ask the questions in small batches, adapt follow-ups to answers, don't dump all of them at once.** Use the AskUserQuestion tool where it helps (multi-select for source lists). Keep it to ~5 minutes. Then synthesize the plan.

### The quiz (six dimensions)

**1. Who are you building for, and what do you build?**
Role (PM / founder / designer / researcher / eng) and the product/feature area. This sets what "a user" means and which segments matter.

**2. Who are your users / customers?**
Segments, personas, or named accounts. The plan will tag sources by which segment they illuminate, and flag segments with no source coverage (a gap worth knowing).

**3. Where does direct user voice live?** *(highest-value — probe hard)*
Multi-select, then ask which they can actually access/export:
- User-interview recordings or transcripts (Gong, Grain, Fathom, Zoom, otter, raw notes)
- Customer-support tickets / conversations (Zendesk, Intercom, Front, email)
- Sales-call notes / CRM (Gong, Salesforce, HubSpot)
- Survey free-text responses (Typeform, Qualtrics, in-app)
- App-store / product reviews
- Community / forum / Discord / Slack-with-customers
- Session-replay or analytics readouts (often *about* users, not their voice — note the distinction)

**4. Where does secondhand / synthesized knowledge live?** *(medium-value)*
Research repositories (Dovetail, Condens, Notion/Confluence research hubs), past readouts, persona docs, JTBD docs, win/loss analyses. Useful, but the lens will mark relayed summaries below firsthand voice — say so.

**5. Where does internal opinion masquerade as user knowledge?** *(low-value — name it so they can deprioritize)*
Strategy decks, "what users want" slides, PRDs full of assumptions, internal Slack debates. Worth ingesting only to *contrast* against real evidence — the lens will grade most of it `discount`.

**6. Access, format, and freshness:**
For their top sources — can they export it (and how: API, CSV, copy-paste, file download)? What format (text, audio needing transcription, PDF)? How fresh does it need to be, and how often does it change? Anything sensitive/permission-gated they should keep out of a shared place?

## Producing the ingestion plan

Synthesize answers into a written plan with these sections. **Customize every line to what they told you** — cite their actual tools and segments.

### `braintrust/sources/` layout
Propose a concrete local folder structure tailored to their sources, e.g.:
```
braintrust/sources/
  interviews/        # transcripts — highest signal
  support/           # exported tickets
  sales-calls/
  surveys/
  reviews/
  research-synthesis/  # secondhand: readouts, personas (mostly borderline)
  _internal/           # opinion decks — ingest last, mostly for contrast
```

### Prioritized source table
Rank by expected evidence value (firsthand+specific first). For each row:

| Priority | Source | Segment(s) it covers | Expected lens verdict | How to get it in | Format / prep | Refresh |
|---|---|---|---|---|---|---|

- **Expected lens verdict** — use the same three-way vocabulary the `evidence-lens` skill outputs, so setup and lens speak one language:
  - **mostly usable** — firsthand interviews, support tickets, observed behavior, bug reports (specific lived experience).
  - **mostly borderline** — secondhand synthesis, surveys, persona/JTBD docs (real but relayed/generalized — corroborate before relying).
  - **mostly discount** — internal opinion decks, "what users want" slides, PRD assumptions (no user behind them).
  Set this expectation per source so the user isn't surprised when the lens grades a deck `discount`.
- **How to get it in:** the concrete export step (e.g. "Gong → export transcript .txt per call", "Zendesk → CSV of last 90 days", "Dovetail → paste highlights").
- **Format / prep:** flag anything needing transcription (audio/video) or extraction (PDF) before it's text.
- **Refresh:** one-time pull vs. recurring, and rough cadence.

### Coverage & gaps
- Which **segments/personas have no source** (you can't learn about users you have no data on).
- Which sources are **opinion-heavy** and will mostly grade `discount` — set expectations so they're not surprised.
- The single **highest-leverage first pull** — the one source most likely to be full of firsthand, specific user behavior. Tell them to start there.

### First step
End with one concrete action that hands off to the lens: *"Create `braintrust/sources/interviews/`, drop your last 5 interview transcripts in as .txt, then run `/braintrust:evidence-lens` on them to see your **usable / borderline / discount** split. Start with the highest-leverage source above — if even that comes back mostly borderline/discount, that's the signal you need more firsthand user voice before building on it."*

## Tone

You're a research-ops co-pilot, not a form. Make recommendations, not an exhaustive survey — if they're light on direct user voice, say plainly that the plan will be evidence-poor until they capture some, and suggest the cheapest way to start (e.g. export support tickets, which they almost always already have). The honest goal is to get *real user signal* into reach of their agent, not to maximize volume.
