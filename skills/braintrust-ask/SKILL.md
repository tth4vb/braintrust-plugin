---
name: braintrust-ask
description: Pressure-test a product idea against your real user research — are users actually asking for or experiencing this? Reads the project's Braintrust running analysis and returns a verdict per hypothesis with specific cited quotes, provenance, and what evidence is MISSING. Use when the user asks "do users want X", "is there evidence for this idea", "should we build this", "what does our research say about <idea>", or wants to check a PR / feature / PRD / design / ticket against real user signal before building.
arguments:
  - name: idea
    description: The idea/feature/change to test — pasted text, a file path, a URL, or an MCP resource ref (Linear/Confluence/Figma/GitHub).
    required: true
---

# Braintrust — ask the evidence

Judge whether the project's accumulated user research actually supports an idea, and surface the
specific quotes that bear on it. This reads what `/braintrust:braintrust-ingest` built; it does not
grade new sources.

**Zero backend.** You (the session) read the local analysis with Read / WebFetch / Bash. Everything
is under `${CLAUDE_PROJECT_DIR}/braintrust/`.

`$ARGUMENTS` (or `$idea`) is the idea to test.

## Step 1 — Preconditions

Require `${CLAUDE_PROJECT_DIR}/braintrust/analysis/digest.md`. If it's missing, tell the user to run
`/braintrust:braintrust-ingest <source>` first to build an evidence base, and stop. Otherwise read
`analysis/manifest.json` and print a freshness line: `Evidence base: <N> claims across <M> sources,
last ingested <last_ingest>.` So the user knows how current the answer is.

## Step 2 — Read the rubric

Read `${CLAUDE_PLUGIN_ROOT}/reference/evidence-lens.md` so your verdict vocabulary
(usable / borderline / discount) matches how the evidence was graded.

## Step 3 — Resolve the idea input (four types)

Same four ways as ingest: local file → Read; URL → WebFetch; MCP resource (Linear issue, Confluence
page, Figma file, GitHub PR) → the connected MCP read tool (if none connected, ask for paste/file/URL);
otherwise treat `$ARGUMENTS` as the pasted idea.

## Step 4 — Extract demand hypotheses

Reduce the idea to **1–5 atomic demand hypotheses** and restate them so the matching is auditable.
A hypothesis is a testable claim about users, e.g. *"users want automated CSV cleanup"*, *"users
will pay a premium for this"*, *"this pain is frequent enough to matter."* Separate the **pain** from
the **proposed solution** from the **willingness to pay** — they're usually evidenced differently.

## Step 5 — Retrieve relevant evidence

- Read `analysis/digest.md` **first** and map each hypothesis to its themes / demand signals.
- For each matched theme, pull the backing claim ids and read **just those lines** from
  `analysis/ledger.jsonl` (grep by id / theme slug / keyword). Don't load the whole ledger.
- Expand a quote's surrounding context from `braintrust/sources/` **only if** a verdict hinges on it.
- If a hypothesis matches no theme in the digest, keyword-search the ledger before concluding
  "no evidence."

## Step 6 — Judge demand, weighted by the gate

For each hypothesis, weigh supporting vs. contradicting claims, weighting
**`usable` >> `borderline` >> `discount`**. The critical rule: a hypothesis "supported" **only** by
discount-tier claims (opinion, hypotheticals, compliments) is **NOT supported** — say so explicitly.
That mismatch — a confident idea resting on weak evidence — is the highest-value thing this surfaces.

Verdict per hypothesis: **supported** (real usable evidence) / **mixed** / **unsupported** (only
weak evidence) / **no-evidence** (nothing in the base speaks to it).

**Then surface the uncertain middle in a "you decide" bucket — never silently count it as support, nor
drop it.** For each hypothesis, pull the relevant `confidence:"torn"` / borderline claims and any
**convergent** weak cluster the digest flags (≥3 segments), and present them verbatim for the user to
judge in their own context. This does **not** change the verdict: a hypothesis backed only by discount
claims is still **unsupported** — but if a cross-segment convergent cluster speaks to it, show it as a
you-decide line rather than omitting it, because the user is the expert on their situation and a
5-segment convergence may matter to them even when it's weak-as-evidence.

## Step 7 — Output

Per hypothesis, then overall:

```
## Idea: <restated in one line>
Evidence base: <N> claims, <M> sources, last ingested <date>.

### Hypothesis 1: <restated>
Verdict: SUPPORTED (strong) — <k> usable claims, <j> contradicting.
- usable · "<verbatim quote>" — <source_file> (<claim id>)
- usable · "<verbatim quote>" — <source_file> (<claim id>)
Provenance: <#sources>, <segment(s)>.

### Hypothesis 2: <restated>
Verdict: NO-EVIDENCE / weak — only <…> discount claims (e.g. "I'd pay for that" = future promise).
Judge unsure — your call in context:
- torn (<committed>/<bandAlt>) · "<verbatim quote>" — <source_file> (<id>) · why: <tornReason>
- convergent-but-weak · <K> hypothetical claims across <S> segments say: "<the shared line>"

## Overall verdict
<Build-readiness in plain language: which parts are well-evidenced (usable), which rest on weak
evidence or nothing. Separate pain / solution-shape / willingness-to-pay.>

## What's MISSING (gaps) — required
- <e.g. no firsthand evidence on willingness to pay>
- <e.g. no coverage of the enterprise segment for this theme>
- Suggested next pull: /braintrust:braintrust-ingest <a source that would close the biggest gap>
```

The **gaps section is required** — naming what evidence is *absent* is as valuable as what's present,
and it tells the user exactly what to go capture.

## Honesty & limits

- The answer is only as current as the **last ingest** (printed in Step 1). It is not live.
- **Absence of evidence is not evidence of absence** — "no-evidence" means the base is silent, not
  that users don't want it. Say which.
- Don't inflate a verdict to be encouraging. A well-grounded "the pain is real but the solution and
  pricing are unevidenced" is more useful than a false green light.
