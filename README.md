# Braintrust — Claude Code plugin

Turn your scattered user research into an **evidence-graded running analysis**, then pressure-test product ideas against it — so your coding agent builds from real user signal instead of opinion.

## The workflow (zero backend — runs in your session)

```
setup  →  ingest  →  ask
```

- **`/braintrust:braintrust-setup`** — a short interview about where your user knowledge lives, ending in a ranked shortlist of sources to feed to ingest. ("What do I even have?")
- **`/braintrust:braintrust-ingest <folder | file | URL | resource>`** — copies the source into a project-local `braintrust/` store, grades **every claim** as **usable / borderline / discount** with a calibrated *Mom Test* evidence lens, and builds a persistent **running analysis** that accumulates across every ingest. Re-run it anytime to add more.
- **`/braintrust:braintrust-ask "<a feature / PR / PRD / idea>"`** — judges whether your real research supports the idea: are users actually asking for or experiencing this? Returns a per-hypothesis verdict **weighted by evidence strength**, with **specific cited quotes + provenance**, and — critically — **what evidence is missing**.

## What it builds (project-local, gitignored)

```
your-repo/
  braintrust/
    .gitignore        # ignores both subdirs — verbatim user quotes never enter git
    sources/          # copies of what you ingested
    analysis/
      ledger.jsonl    # every graded claim (quote, verdict, provenance)
      manifest.json   # what's been ingested (dedup + counts)
      digest.md       # synthesized analysis: demand signals by evidence strength + gaps
```

The evidence base stays **local** — `ask` reads `digest.md` for the overview and dips into the ledger for exact quotes.

## Install

```bash
/plugin marketplace add anthropics/claude-plugins-community
/plugin install braintrust@claude-community
```

Then:

```bash
/braintrust:braintrust-setup                          # see what sources you have
/braintrust:braintrust-ingest ~/research/interviews   # grade them, build the base
/braintrust:braintrust-ask "should we build bulk CSV import?"   # test an idea
```



## Development

Bare plugin (manifest at `.claude-plugin/plugin.json`, skills under `skills/`, shared rubric in `reference/`), referenced by any marketplace via `source: {github, repo: "tth4vb/braintrust-plugin"}`.

- Validate: `claude plugin validate .`
- Local install test: a gitignored `dev-marketplace/` is provided — `/plugin marketplace add ./dev-marketplace` then `/plugin install braintrust@braintrust-dev`.

## License

MIT
