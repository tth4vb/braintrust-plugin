# Brain Trust — Claude Code plugin

Put your company's scattered knowledge through a **user-research evidence lens**, so your coding agent builds from real user signal instead of opinion.

This repo is a single Claude Code **plugin** (`braintrust`). It's distributed through the official community marketplace — see Install.

## What's in v0.1 (zero-backend, ships today)

Two skills, no servers, no ingestion infrastructure — the lens runs entirely in your Claude Code session:

- **`/braintrust:evidence-lens`** — grades any user-research text (interview notes, support tickets, survey answers, sales-call notes, "what users want" claims) as **usable / borderline / discount**, applying a calibrated *Mom Test* rubric. Surfaces which conclusions rest on weak evidence. Validated on a public held-out set with **zero hard usable↔discount errors**; flags the genuinely marginal middle as *borderline* instead of guessing.
- **`/braintrust:braintrust-setup`** — a short interview about where your user knowledge lives, ending in a **customized ingestion plan**: a prioritized list of which sources to pull into a local `braintrust/sources/` folder, why, and how — with each source tagged by its expected lens verdict.

## Install

```bash
# add the official community marketplace, then install
/plugin marketplace add anthropics/claude-plugins-community
/plugin install braintrust@claude-community
```

Then:

```bash
/braintrust:braintrust-setup        # plan what to pull in
/braintrust:evidence-lens           # grade what you've got
```

## The idea

`setup` decides *what knowledge to gather*; `evidence-lens` decides *what's trustworthy once it's there*. The lens biases toward **firsthand, specific user behavior** (usable evidence) over opinion, hypotheticals, and compliments (the fool's gold of customer learning), and says *borderline* when a signal is real-but-relayed or only thinly behavioral.

## Roadmap (not in v0.1)

The full Brain Trust is a personal tool that **ingests** from local folders + your own Google Drive, runs every chunk through the lens at ingest, and serves the **evidence-graded** slice to coding agents via an **MCP server** + a `pull` context-pack. That backend isn't shipped here — v0.1 is the validated lens + setup planner, the cheapest way to find out if the lens is useful before building ingestion. When the backend exists, it plugs in as an `mcpServers` entry + a `/braintrust:pull` command in this same plugin.

## Development

This repo is a bare plugin (manifest at `.claude-plugin/plugin.json`, skills under `skills/`), so it can be referenced directly by any marketplace via `source: {github, repo: "tth4vb/braintrust-plugin"}`.

- Validate: `claude plugin validate .`
- Local install test: a gitignored `dev-marketplace/` is provided — `/plugin marketplace add ./dev-marketplace` then `/plugin install braintrust@braintrust-dev`.

## License

MIT
