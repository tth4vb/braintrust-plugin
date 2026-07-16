# Proposal — surfacing uncertainty to the user ("judge unsure → you decide")

**Status:** ✅ IMPLEMENTED 2026-07-16 (plugin v0.3.0) — both skills + templates + manifest updated on `main`; the real
`../ff-scaffold/braintrust/analysis/digest.md` was regenerated end-to-end to demonstrate it. · **Drafted:** 2026-07-16
**Decisions:** convergence = ≥3 distinct segments; torn = structural triggers only (v1).

## The idea
When the lens is unsure, **disclose the evidence and let the user judge it in the context of their
decision** — instead of silently committing a band and burying the quote. The prototype on real data
showed this needs **two** mechanisms, not one:

- **A — legible torn-ness** on the marginal middle (fires on 6/58, all in the borderline pile, zero
  false positives). Makes the *already-happening* "lift a real pain up from discount" tie-break visible.
- **B — convergence over the weak pile** (the bigger, less-obvious win): 7 individually-fluff WTP
  hypotheticals across 5 segments draw one line — *"pay for the doing, not the knowing"* — currently
  buried in the discount pile. Surface clustered **user** signal for you-decide, without re-grading it.

**Discipline up front:** neither mechanism changes any rubric axis, weight, or cutoff. `evidence-lens.md`
is untouched → **no calibration gate**, no rubric-version bump. This is a **plugin/skill** change
(bump `plugin.json`, not the rubric).

---

## Mechanism A — `confidence` on each claim

### A1. Ledger schema delta (`braintrust-ingest` Step 7)
Add three optional fields to the per-claim record:
```jsonc
"confidence": "clear" | "torn",   // default "clear"; omit or "clear" == today's behavior
"bandAlt":    "usable|borderline|discount",  // the runner-up band, ONLY when torn
"tornReason": "<one line: why the call wobbled, in Mom-Test terms>"
```
Back-compatible: absent = `clear`. Existing ledgers need **no re-ingest** — `torn` is derivable from
fields already present (see A2), so a digest regen back-fills it.

### A2. When to emit `torn` (reserve it — over-flagging kills the signal)
**DECIDED 2026-07-16 — structural triggers only** (deterministic, back-derivable over existing ledgers;
fired 6/58 with zero false positives on ff-scaffold). Judge self-report is a possible later addition,
not part of v1. Emit `confidence: "torn"` **only** for a genuine *adjacent-band* tie, triggered by:
1. `compliment` flag on a **fact-tier** concreteness — the fact survived the multi-claim rule but rests
   on a praise-laden quote (`bandAlt` = borderline).
2. `leading_question` present — the answer was pulled, but may embed a real current pain
   (`bandAlt` = the band on the other side of the committed call).
3. `secondhand_summary × general_pattern` (~0.33) — real-but-relayed (`bandAlt` = usable *or* discount
   depending on how concrete the relayed thing is).
4. **Judge-lift:** formula scores `discount` (< 0.30) but a concrete, *current* pain sits under a wish,
   so the judge commits `borderline` (`bandAlt` = discount). *(This is 4 of the 6 in ff-scaffold — the
   "err toward disclosure" tie-break, now recorded instead of implicit.)*
5. Genuinely torn between `opinion_or_compliment` and `general_pattern` — the "sliver of implied
   behavior" case the rubric already names (`evidence-lens.md` §"Borderline is a real verdict").

Do **not** emit torn for a confident call whose score merely lands near a cutoff (e.g. a clean
`firsthand × general_pattern = 0.55` is *not* torn — it's a confident usable).

### A3. Tie-break rule
When torn between two adjacent bands, **commit the more-visible band** (borderline over discount) and
set `confidence:"torn"` + `bandAlt`. Rationale: skepticism still governs the **score**; disclosure
governs the **tie-break**. The quote is shown either way — the tie-break only decides *which section and
how prominent*, and the `torn` tag keeps it honest about strength.

---

## Mechanism B — convergence pass over the weak pile

### B1. New sub-step (`braintrust-ingest`, end of Step 8, before writing the digest)
After grading, group **`discount` and `borderline`** claims by theme-family and flag a cluster as
`convergent` when it spans **≥3 distinct `segment`s** (DECIDED 2026-07-16 — segments, not sources, so a
single loud person can't manufacture convergence), scoped to **user-voiced claims only**:
- **Include:** `speaker` ∈ {user, researcher-relaying-user} and `segment` ≠ `internal`.
- **Exclude:** `internal` segment / `teammate` speaker (strategy-deck assertions never convergence-surface).

Emit a lightweight annotation (not a band change): a `convergence` block in `manifest.json` listing
`{theme_family, segments:[...], claim_ids:[...]}`. Claims **stay** `discount`/`borderline`.

### B2. Why the guardrail matters
This is the line between "listening to your users" and "laundering wishful thinking." Five *segments*
independently saying they'll pay for action is signal; the company's own slide saying *"households are
desperate for a neutral tool"* is not — and B1's `internal` exclusion enforces exactly that.

---

## Presentation deltas

### C1. `digest.md` (ingest Step 8 template)
Replace the flat `### Emerging (borderline …)` block with a split, and add a convergence block:
```
### ⚑ Judge unsure — you decide (<T> claims)
_The lens flagged these as marginal — it's handing you the quote + why it wobbled, not a verdict._
**A · real pain, phrased as a wish** (judge lifted from discount)
- "<verbatim>" — <segment> · <source> (<id>) · _why torn: <tornReason>_ · _your call: <the contextual question>_
**B · real, but relayed** (corroborate before building)
- "<verbatim>" — <segment, relaying> · <source> (<id>)

### ◇ Weak as evidence — but it converges, so you weigh it (<K> claims, <S> segments)
_Each is correctly discounted alone; together, across independent segments, they draw one line._
| segment | quote (abridged) | shared signal |
```
`### Strong (usable)` stays unchanged and stays first. `### Weak / asserted (discount)` keeps the
*non-convergent* discounts (incl. all internal assertions) exactly as today.

### C2. `manifest.json` totals delta
`"totals": { …, "torn": <T>, "convergent_clusters": <C> }` — at-a-glance visibility.

### C3. `braintrust-ask` (Steps 6–7)
- Step 6: a hypothesis "supported" only by discount claims is still **unsupported** (unchanged) — **but**
  if a *convergent* discount cluster speaks to it, add a **"you decide"** line rather than silently
  dropping it. Torn/borderline claims are shown verbatim in a you-decide bucket, never counted as
  settled support **nor** omitted.
- Step 7 output, per hypothesis, add when relevant:
```
Judge unsure — your call in context:
- torn (borderline/discount) · "<verbatim>" — <source> (<id>) · why: <tornReason>
- convergent-but-weak · <K> hypothetical claims across <S> segments say: "<the shared line>"
```
The required **What's MISSING** section is unchanged.

---

## Versioning, migration, non-goals
- **Version:** bump `plugin.json` (skills changed). **Rubric untouched** → `evidence-lens.md` version
  stays, no calibration-gate run.
- **Migration:** append-only store is preserved. New fields are additive; `torn` + `convergence` are
  **re-derivable** from existing records, so a plain `digest.md` regen upgrades any existing store with
  no re-grade.
- **Non-goals:** no new bands; no score/cutoff changes; no auto-promotion of weak claims to usable.
  Convergence changes *visibility*, never *band*.

## Decisions (2026-07-16)
1. **Convergence threshold:** ✅ **≥3 distinct segments** (not sources) — a single loud person can't
   manufacture convergence.
2. **Torn detection:** ✅ **structural triggers only** (A2) for v1; judge self-report is a possible
   later addition.

## Open question remaining
3. **Naming:** "⚑ Judge unsure — you decide" and "◇ converges, so you weigh it" — keep as-is unless you
   want different words.
```
