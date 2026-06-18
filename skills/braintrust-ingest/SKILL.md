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
"created":"<now>","last_ingest":null,"sources":[],"totals":{"sources":0,"claims":0,"usable":0,"borderline":0,"discount":0}}`.

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

## Step 7 — Append to ledger.jsonl, update manifest.json

For each new claim, append **one JSON object per line** to `analysis/ledger.jsonl`:
```json
{"id":"clm_<short hash of source_file+quote>","quote":"<verbatim>","source_file":"braintrust/sources/<…>","source_locator":"<lines / heading / timestamp / page, best effort>","speaker":"user|researcher|teammate|unknown","directness":"<axis1>","concreteness":"<axis2>","biasFlags":[],"evidenceScore":0.0,"gate":"usable|borderline|discount","themes":["<slug>"],"segment":"<segment|unknown>","ingest_run":"<iso8601>","rubric_version":"<from step 1>"}
```
Then update `manifest.json`: add/replace the source record (`src_path`, `origin`, `content_hash`,
`bytes`, `claim_count`, `claim_ids`, `ingested_at`), set `last_ingest`, and recompute `totals` from
the per-source counts. Append after each file so a mid-run interruption leaves a consistent store.

## Step 8 — Regenerate digest.md across ALL claims

Rewrite `analysis/digest.md` wholesale from the full ledger (a new source can strengthen or weaken
an existing theme, so the digest must be whole). Read `ledger.jsonl` (compact — not the raw
sources) and produce:

```
# Braintrust analysis digest
_Last updated: <date> · <N> claims across <M> sources · <U> usable / <B> borderline / <D> discount · rubric v<version>_

## Demand signals & pain points (by evidence strength)

### Strong (usable evidence)
- **<theme as a plain-language signal>.** <count> usable claims.
  - "<top verbatim quote>" — <source_file> (<claim id>)
  - Segments: <…>. Flags seen: <…>.

### Emerging (borderline — corroborate before building)
- **<signal>.** <count> borderline claims (real-but-relayed / thinly behavioral).

### Weak / asserted (discount — opinion, hypothetical, no user behind it)
- **<signal>.** <directness/concreteness reason>; do not build on alone.

## Theme index
| theme | usable | borderline | discount | top source |
|---|---|---|---|---|

## Coverage & gaps
- Segments with NO usable evidence: <…>
- Themes asserted only in the discount pile (no usable backing): <…>
```

If claims carry mixed `rubric_version`s, add a one-line warning in the footer.

## Step 9 — Report

Tell the user: N new sources, M new claims, the usable / borderline / discount delta, the single
highest-leverage new theme, and any `needs_prep` files skipped. End by suggesting
`/braintrust:braintrust-ask "<an idea to pressure-test>"`.

## Honesty & limits

- **Scale:** for a large folder, grade file-by-file and append after each — don't try to hold the
  whole corpus in context. The digest is the compaction layer everything else reads.
- **Grading quality depends on the model running this session** — the rubric is calibrated, but
  weaker models produce noisier grades.
- **Nothing usable?** Say so plainly — that's the signal the user needs more firsthand user voice,
  not a reason to inflate grades.
- **Freshness:** the analysis only reflects what's been ingested. It is not live.
