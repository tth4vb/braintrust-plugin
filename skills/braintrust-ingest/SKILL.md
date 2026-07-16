---
name: braintrust-ingest
description: Ingest user-research sources into a project-local braintrust/ evidence store and grade every claim with the calibrated Mom Test evidence lens, building a persistent running analysis. Use when the user wants to "ingest research", "add a transcript / tickets / survey / source", "grade my user research", "pull this into braintrust", "build my evidence base", or points you at a folder, file, URL, or connected resource of customer/user knowledge.
arguments:
  - name: source
    description: What to ingest — a folder path, a file path, a URL, an MCP resource reference, or pasted text.
    required: true
---

# Braintrust — ingest & grade user-research sources

Copy a source of real user knowledge into a project-local evidence store, grade every claim in it
with the Braintrust evidence lens, and update a persistent **running analysis** that accumulates
across every ingest. This is the gather-and-grade step; later, `/braintrust:braintrust-ask` queries
what you build here.

**Zero backend.** You (the session) do all the work with Read / WebFetch / Bash / Write. There is no
server. Everything lives in `${CLAUDE_PROJECT_DIR}/braintrust/`.

`$ARGUMENTS` (or `$source`) is what to ingest.

## Step 1 — Read the rubric (do this first, every run)

Read `${CLAUDE_PLUGIN_ROOT}/reference/evidence-lens.md` and apply it **verbatim** when grading. Do
not paraphrase or re-derive the gate. Note its `Rubric version` — you'll stamp each graded claim
with it.

## Step 2 — Ensure the store exists

Under `${CLAUDE_PROJECT_DIR}/braintrust/`, create if missing:
- `sources/` — copies of ingested files
- `analysis/` — `ledger.jsonl`, `manifest.json`, `digest.md`
- `.gitignore` containing exactly:
  ```
  sources/
  analysis/
  ```
  Both are gitignored on purpose: they embed verbatim user quotes that must never enter git. Confirm
  to the user that nothing under `braintrust/` will be committed.

If `analysis/manifest.json` doesn't exist, initialize it: `{"rubric_version":"<from step 1>",
"created":"<now>","last_ingest":null,"sources":[],"convergence":[],"totals":{"sources":0,"claims":0,"usable":0,"borderline":0,"discount":0,"torn":0,"convergent_clusters":0}}`.

## Step 3 — Resolve the input (four types)

Detect what `$ARGUMENTS` is and read it with whatever the session already has:
- **Local folder** → Read it (recurse; grade text-like files: .txt .md .csv .json .vtt .srt etc.).
- **Local file** → Read it.
- **URL** → WebFetch it.
- **MCP resource ref** (e.g. a Linear issue, Confluence page, Figma file, GitHub PR) → use the
  connected MCP read tool. If no such MCP is connected, say so and ask the user to paste the content
  or give a file/URL — do not invent integration.
- **Pasted text** → treat as one in-memory source named `pasted-<timestamp>`.

Binary / audio / PDF you can't reliably read as text → record it in the manifest as `needs_prep`
and **skip grading it** (tell the user it needs transcription/extraction first).

## Step 4 — Copy into sources/

Copy each readable source into `braintrust/sources/`, preserving a readable subtree: use the
original folder name, or a type slug (`interviews/`, `support/`, `surveys/`, `sales-calls/`). For
URLs / MCP / pasted input, write the fetched text to a sensibly named `.md`/`.txt` and record where
it came from as `origin`.

## Step 5 — Dedup before grading (content-hash primary)

For each candidate source, compute a content hash of its normalized text — run `shasum -a 256` (or
`sha256sum`) via Bash (read-only). Then:
- **Hash already in `manifest.sources`** → skip; report "already ingested, unchanged."
- **Same `src_path`, different hash** → it changed: mark the old source's `claim_ids` as superseded
  and re-grade just that file.
- **New** → grade it (next step).

Normalize whitespace before hashing so trivial reformatting doesn't create near-duplicates.

## Step 6 — Chunk & grade NEW sources only

Apply the rubric from Step 1. Split each source into individual claims/quotes, grade each on
directness + concreteness + bias flags, compute `evidenceScore`, and assign the `gate`
(usable / borderline / discount). **Never re-read or re-grade existing ledger claims** — that's the
point of an append-only store; only new (or changed) sources are graded.

**Then set `confidence` and apply the disclosure tie-break.** When the lens is genuinely torn between
two *adjacent* bands, don't silently pick — surface the wobble so the user can judge the quote in the
context of their own decision. A claim is `torn` when ANY of these structural triggers holds:
- `compliment` flag on a **fact-tier** concreteness (the fact survived the multi-claim rule, but rests
  on a praise-laden quote) → `bandAlt: borderline`;
- `leading_question` present (a pulled answer that may still embed a real *current* pain) → `bandAlt` =
  the band on the other side of the committed call;
- `secondhand_summary` × `general_pattern` (real-but-relayed) → `bandAlt` = usable or discount, by how
  concrete the relayed thing is;
- a formula score < 0.30 that you **lift** to `borderline` because a concrete, current pain sits under a
  wish → `bandAlt: discount`;
- genuinely torn between `opinion_or_compliment` and `general_pattern` (a sliver of implied behavior) →
  `bandAlt` = the other.

**Tie-break:** when torn, commit the **more-visible** band (borderline over discount) and set
`confidence: "torn"`. Skepticism still governs the *score*; disclosure governs the *tie-break* — the
quote surfaces either way, and the `torn` tag keeps it honest about strength. Everything else is
`confidence: "clear"`. Do **not** mark a confident call torn just because its score lands near a cutoff
(a clean `firsthand_stated × general_pattern = 0.55` is a confident usable, not torn).

## Step 7 — Append to ledger.jsonl, update manifest.json

For each new claim, append **one JSON object per line** to `analysis/ledger.jsonl`:
```json
{"id":"clm_<short hash of source_file+quote>","quote":"<verbatim>","source_file":"braintrust/sources/<…>","source_locator":"<lines / heading / timestamp / page, best effort>","speaker":"user|researcher|teammate|unknown","directness":"<axis1>","concreteness":"<axis2>","biasFlags":[],"evidenceScore":0.0,"gate":"usable|borderline|discount","confidence":"clear|torn","bandAlt":"<runner-up band — ONLY when torn>","tornReason":"<one line in Mom-Test terms — ONLY when torn>","themes":["<slug>"],"segment":"<segment|unknown>","ingest_run":"<iso8601>","rubric_version":"<from step 1>"}
```
Omit `bandAlt`/`tornReason` (or leave `confidence:"clear"`) for non-torn claims — that's the default and
matches pre-existing ledgers.

Then update `manifest.json`: add/replace the source record (`src_path`, `origin`, `content_hash`,
`bytes`, `claim_count`, `claim_ids`, `ingested_at`), set `last_ingest`, and recompute `totals` (including
`torn` and `convergent_clusters`) from the per-source counts. Append after each file so a mid-run
interruption leaves a consistent store.

## Step 8 — Detect convergence, then regenerate digest.md across ALL claims

**First, detect convergence in the weak pile.** A single hypothetical is fluff; the *same* weak signal
voiced independently across many segments is worth putting in front of the user. Group `discount` +
`borderline` claims by theme-family and flag a cluster as **convergent** when it spans **≥3 distinct
`segment`s**, counting **user-voiced claims only** — `segment` ≠ `internal` and `speaker` ≠ `teammate`.
Record each cluster in `manifest.json` under `convergence` as `{theme_family, segments:[…], claim_ids:[…]}`,
and set `totals.convergent_clusters`. **Internal / strategy-deck assertions never convergence-surface** —
that's the line between listening to users and laundering the company's own wishful thinking.

**Then rewrite `analysis/digest.md` wholesale** from the full ledger (a new source can strengthen or
weaken an existing theme, so the digest must be whole). Read `ledger.jsonl` (compact — not the raw
sources) and produce:

```
# Braintrust analysis digest
_Last updated: <date> · <N> claims across <M> sources · <U> usable / <B> borderline / <D> discount · <T> torn · rubric v<version>_

## Demand signals & pain points (by evidence strength)

### Strong (usable evidence)
- **<theme as a plain-language signal>.** <count> usable claims.
  - "<top verbatim quote>" — <source_file> (<claim id>)
  - Segments: <…>. Flags seen: <…>.

### ⚑ Judge unsure — you decide (<count> claims)
_The lens flagged these as marginal — it hands you the quote and why it wobbled, NOT a verdict. Judge
them in the context of your decision. (borderline claims + any `confidence:"torn"`.)_
**A · real pain, phrased as a wish** (lifted from discount)
- "<verbatim>" — <segment> · <source_file> (<id>) · _why: <tornReason>_ · _your call: <the contextual question this raises>_
**B · real, but relayed** (corroborate before building)
- "<verbatim>" — <segment, relaying> · <source_file> (<id>)

### ◇ Weak as evidence — but it converges, so you weigh it (<K> claims across <S> segments)
_Each is correctly discounted alone; together, across ≥3 independent segments, they draw one line. Still
not `usable` — you decide what a cross-segment convergence is worth._
- **<the shared line, in plain language>** — <S> segments:
  | segment | quote (abridged) |
  |---|---|
  | <segment> | "<verbatim, abridged>" |

### Weak / asserted (discount — opinion, hypothetical, no user behind it)
- **<signal>.** <directness/concreteness reason>; do not build on alone.
  _(Non-convergent discounts, including all internal/strategy assertions, stay here with no you-decide surface.)_

## Theme index
| theme | usable | borderline | discount | top source |
|---|---|---|---|---|

## Coverage & gaps
- Segments with NO usable evidence: <…>
- Themes asserted only in the discount pile (no usable backing): <…>
```

Keep **Strong (usable)** first and undiluted. If there are no torn/borderline claims, omit the "Judge
unsure" section; if no convergent cluster, omit the convergence section — don't print empty headers.
If claims carry mixed `rubric_version`s, add a one-line warning in the footer.

## Step 9 — Report

Tell the user: N new sources, M new claims, the usable / borderline / discount delta, the single
highest-leverage new theme, any new **convergent** weak cluster (the "you decide" signal), and any
`needs_prep` files skipped. End by suggesting `/braintrust:braintrust-ask "<an idea to pressure-test>"`.

## Honesty & limits

- **Scale:** for a large folder, grade file-by-file and append after each — don't try to hold the
  whole corpus in context. The digest is the compaction layer everything else reads.
- **Grading quality depends on the model running this session** — the rubric is calibrated, but
  weaker models produce noisier grades.
- **Nothing usable?** Say so plainly — that's the signal the user needs more firsthand user voice,
  not a reason to inflate grades.
- **Freshness:** the analysis only reflects what's been ingested. It is not live.
