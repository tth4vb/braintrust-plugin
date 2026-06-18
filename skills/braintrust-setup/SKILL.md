---
name: braintrust-setup
description: Help the user see what company-knowledge sources they could use as user-research evidence. A short interview about where their customer knowledge lives, ending in a ranked shortlist of sources to feed to braintrust-ingest. Use when the user wants to "set up Braintrust", "figure out what to ingest", "what knowledge could I use as evidence", "onboard my research sources", or is starting to build an evidence base for an agent.
---

# Braintrust Setup — find your evidence sources

Goal: in one short interview, help the user **see where their real user-knowledge lives** and hand them a **ranked shortlist of sources** to feed to `/braintrust:braintrust-ingest`. This is the "what do I even have?" step — it does no grading itself; ingest does that. Shape the shortlist to their answers; don't produce a generic checklist.

Part of the workflow: **setup** (see what you have) → **`braintrust-ingest`** (copy it in, grade every claim, build the running analysis) → **`braintrust-ask`** (pressure-test ideas against it). Bias the shortlist toward sources likely to contain **firsthand, specific user behavior** (the lens's "usable" tier) over opinion/secondhand sources.

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

## Producing the source shortlist

Synthesize answers into a written shortlist with these sections. **Customize every line to what they told you** — cite their actual tools and segments. This is *what to feed ingest*, not a folder you build by hand — `braintrust-ingest` creates and manages `braintrust/sources/` + `braintrust/analysis/` for them.

### Prioritized source table
Rank by expected evidence value (firsthand+specific first). For each row:

| Priority | Source | Segment(s) it covers | Expected lens verdict | How to get it in | Format / prep | Refresh |
|---|---|---|---|---|---|---|

- **Expected lens verdict** — use the same three-way vocabulary `braintrust-ingest` will assign, so setup and ingest speak one language:
  - **mostly usable** — firsthand interviews, support tickets, observed behavior, bug reports (specific lived experience).
  - **mostly borderline** — secondhand synthesis, surveys, persona/JTBD docs (real but relayed/generalized — corroborate before relying).
  - **mostly discount** — internal opinion decks, "what users want" slides, PRD assumptions (no user behind them).
  Set this expectation per source so the user isn't surprised when the lens grades a deck `discount`.
- **How to get it to ingest:** the concrete step to make it ingestible (e.g. "Gong → export transcript .txt per call", "Zendesk → CSV of last 90 days, then `braintrust-ingest <that file>`", "Dovetail → paste highlights", "a public doc → just pass the URL"). `braintrust-ingest` accepts a folder, file, URL, or MCP resource.
- **Format / prep:** flag anything needing transcription (audio/video) or extraction (PDF) before it's text — ingest will mark those `needs_prep` and skip them.
- **Refresh:** one-time pull vs. recurring re-ingest, and rough cadence.

### Coverage & gaps
- Which **segments/personas have no source** (you can't learn about users you have no data on).
- Which sources are **opinion-heavy** and will mostly grade `discount` — set expectations so they're not surprised.
- The single **highest-leverage first source** — the one most likely to be full of firsthand, specific user behavior. Tell them to start there.

### First step
End with one concrete action that hands off to ingest: *"Start with your highest-leverage source above and run `/braintrust:braintrust-ingest <path-or-url>`. It copies it into a gitignored `braintrust/` store, grades every claim usable / borderline / discount, and builds your running analysis. Then `/braintrust:braintrust-ask "<an idea>"` to pressure-test a feature against it. If even your best source comes back mostly borderline/discount, that's the signal you need more firsthand user voice before building."*

## Tone

You're a research-ops co-pilot, not a form. Make recommendations, not an exhaustive survey — if they're light on direct user voice, say plainly that the plan will be evidence-poor until they capture some, and suggest the cheapest way to start (e.g. export support tickets, which they almost always already have). The honest goal is to get *real user signal* into reach of their agent, not to maximize volume.
