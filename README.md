# Brain Trust — Claude Code plugin

Put your company's scattered knowledge through a **user-research evidence lens**, so your coding agent builds from real user signal instead of opinion.

This repo is both a **plugin marketplace** and the **`braintrust` plugin** it serves.

## What's in v0.1 (zero-backend, ships today)

Two skills, no servers, no ingestion infrastructure — the lens runs entirely in your Claude Code session:

- **`/braintrust:evidence-lens`** — grades any user-research text (interview notes, support tickets, survey answers, sales-call notes, "what users want" claims) as **usable** vs. **discount**, applying a calibrated *Mom Test* rubric. Surfaces which conclusions are resting on weak evidence. Calibrated to ~94% agreement with human judgment on a 33-item gold set.
- **`/braintrust:braintrust-setup`** — a short interview about where your user knowledge lives, ending in a **customized ingestion plan**: a prioritized list of which sources to pull into a local `braintrust/sources/` folder, why, and how.

## Install

```bash
# add this marketplace (local path, or your git remote once pushed)
/plugin marketplace add ./braintrust-plugin
# or: /plugin marketplace add <your-github-user>/braintrust-plugin

/plugin install braintrust@braintrust
```

Then:

```bash
/braintrust:braintrust-setup        # plan what to pull in
/braintrust:evidence-lens           # grade what you've got
```

## The idea

`setup` decides *what knowledge to gather*; `evidence-lens` decides *what's trustworthy once it's there*. The lens biases toward **firsthand, specific user behavior** (strong evidence) over opinion, hypotheticals, and compliments (the fool's gold of customer learning).

## Roadmap (not in v0.1)

The full Brain Trust is a personal tool that **ingests** from local folders + your own Google Drive, runs every chunk through the lens at ingest, and serves the **evidence-graded** slice to coding agents via an **MCP server** + a `pull` context-pack. That backend isn't shipped here — v0.1 is the validated lens + setup planner, the cheapest way to find out if the lens is useful before building ingestion. When the backend exists, it plugs in as an `mcpServers` entry + a `/braintrust:pull` command in this same plugin.

## License

MIT
