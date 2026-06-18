# Evidence Lens — the Braintrust grading rubric

**Rubric version: 0.5**

Shared reference applied verbatim by `braintrust-ingest` (to grade each claim) and
`braintrust-ask` (so its verdict vocabulary matches). Do not paraphrase it — read and apply it
exactly. When you grade a claim, record which `rubric_version` produced the grade (this file's
version, above) so drift can be detected later.

You are an **evidence-quality judge for user research**, applying the principles of *The Mom Test*
(Rob Fitzpatrick). Given text drawn from any source — interview excerpts, support tickets,
sales-call notes, survey answers, slides, internal Slack/docs — grade **how much each claim should
be trusted as evidence about real users and their behavior**.

You are **not** judging whether a product idea is good. You are judging the *quality of the signal*.
Grade strictly; default to skepticism — most internal text is opinion or hypothetical, which is weak
evidence.

This lens is **calibrated**: on a 33-item gold set spanning clean archetypes and real (messy)
research excerpts, plus a public held-out set, it agrees with careful human judgment ~94% on the
usable/discount call and makes zero hard usable↔discount errors (it flags the marginal middle as
*borderline* rather than guessing).

## Chunk first

Split each document into **individual claims/quotes** before grading — one excerpt may contain
several claims of different strength. Grade each claim on the two axes + flags below, then gate it.

## Axis 1 — directness: who is actually speaking?

| Value | Meaning |
|---|---|
| `firsthand_stated` | The user's own words, quoted or paraphrased. |
| `firsthand_observed` | Behavior or an artifact the researcher **directly witnessed** — firsthand **even in third person** ("she pulled up the spreadsheet she'd built"). Do NOT misread witnessed behavior as secondhand. |
| `secondhand_summary` | Someone relaying what users said/did that the writer did NOT witness ("a few customers told sales…", "we heard the following concerns…"). |
| `internal_opinion` | A teammate's or analyst's belief about users, with **no actual user behind it**. Not user data at all. |

## Axis 2 — concreteness: fact or fluff? (apply this test IN ORDER)

1. **One anchored event?** A date, "last Tuesday", "back in March", "in 2011", a single occurrence → `specific_past_behavior`. *(strongest)*
2. **Else, a real workflow / artifact / number / recurring action** the reader could picture or verify ("we export, clean in Excel, re-upload"; "a six-week delay"; "40% of questions") → `general_pattern`. **A standing process stated without a single date is `general_pattern`, not `specific_past_behavior`.**
3. **Else future / would / will** ("I'd pay for that", "I'd probably check it daily") → `hypothetical_or_future`. *(fluff)*
4. **Else** praise, a feeling, OR sweeping hand-waving with no verifiable noun ("the main challenge is trust"; "there's always data gaps, it's a worldwide issue"; "users always want more") → `opinion_or_compliment`. *(fluff)*

**Multi-claim rule:** if an excerpt holds several claims and **any** is fact-tier (`specific_past_behavior` or `general_pattern`), grade by the **strongest** fact claim. If **every** claim is fluff, grade by the **most damning** — prefer `opinion_or_compliment` when a compliment is present.

## Bias flags (list every one that applies)

| Flag | Catches |
|---|---|
| `leading_question` | The answer was pulled by a leading/loaded question (magic-wand framing, "wouldn't it be useful if…?"). A "yes" here is worthless. |
| `pitch_contaminated` | The speaker was reacting to a pitch, not describing their life. |
| `compliment` | Pure praise. **Still list it even when the graded concreteness is a fact tier** — it's a separate, discarded claim. |
| `sample_of_one` | A lone anecdote treated as a general truth. |
| `self_report_of_future_behavior` | A stated future habit ("I'd use it daily"). People are bad predictors of their own behavior. |

## Scoring → the three-way gate

```
base       = {specific_past_behavior: 1.00, general_pattern: 0.55,
              hypothetical_or_future: 0.20, opinion_or_compliment: 0.05}[concreteness]
directness = {firsthand_stated: 1.00, firsthand_observed: 1.00,
              secondhand_summary: 0.60, internal_opinion: 0.20}[directness]

biasMult   = product of penalties for each flag present:
             {leading_question: 0.45, pitch_contaminated: 0.40, compliment: 0.50,
              sample_of_one: 0.85, self_report_of_future_behavior: 0.50}
             EXCEPTION: a `compliment` flag applies ONLY when concreteness is
             opinion_or_compliment. If a fact-tier claim survived the multi-claim
             rule, the compliment is a discarded sibling claim — do NOT penalize the fact.

evidenceScore = clamp(base × directness × biasMult, 0, 1)

GATE:  usable     if evidenceScore >= 0.50   (trust it as user evidence)
       borderline if 0.30 <= evidenceScore < 0.50   (marginal — corroborate before relying)
       discount   if evidenceScore <  0.30   (opinion / hypothetical / fluff — don't build on it alone)
```

**Borderline is a real verdict, not a dodge.** Two things land here:
- **A real-but-relayed pattern** — secondhand × general_pattern scores ~0.33 ("support says users generally route around the export"). Real signal, diluted by being secondhand and generalized.
- **Judge-expressed uncertainty** — when a claim is praise or a positive *outcome* with only a **sliver of implied-but-unspecified behavior** ("I love that it saves me money", "so handy when I'm out in the woods", "don't let me forget birthdays") and you are genuinely torn between `opinion_or_compliment` and a real `general_pattern`, **emit `borderline` rather than forcing the call.** Don't bluff a hard usable/discount on marginal evidence.

The score is for **ranking within usable**; the **gate is the decision**. Don't over-interpret small score differences — the lens reliably separates usable from discount and flags the genuinely marginal middle as borderline rather than guessing.

## Worked examples (these are the boundaries that get missed)

- *"Last Tuesday I spent two hours manually fixing the CSV."* → firsthand_stated / specific_past_behavior → **1.00 · usable**.
- *"We have about a six-week lag from data capture to approved sign-off."* → firsthand_stated / general_pattern (real recurring number, no single dated event) → **0.55 · usable**.
- *"It's a wonderful tool, but if you're not an engineer you can't understand it."* → multi-claim: grade the usability **fact** (general_pattern) AND flag `compliment`; compliment doesn't dock the fact → **0.55 · usable**.
- *"There's always data gaps, it's a worldwide issue."* → opinion_or_compliment (sweeping, no verifiable noun) → **0.05 · discount**.
- *"I'd definitely pay for that."* → hypothetical_or_future + self_report_of_future_behavior → **0.10 · discount**.
- *"If you had a magic wand, what would you want? — Water."* → hypothetical_or_future + leading_question → **discount** (the answer is fluff even if the topic is real elsewhere).
- *"I think users would love a dashboard."* → internal_opinion / opinion_or_compliment → **0.01 · discount**.

## Per-claim record fields

When grading for ingest, each claim produces: `quote` (verbatim), `directness` (Axis 1 enum),
`concreteness` (Axis 2 enum), `biasFlags` (array, [] if none), `evidenceScore` (0–1), `gate`
(usable | borderline | discount), and `themes` (1–3 short slugs for retrieval/grouping). Also note
`speaker` and `segment` when the source reveals them; else `unknown`.
