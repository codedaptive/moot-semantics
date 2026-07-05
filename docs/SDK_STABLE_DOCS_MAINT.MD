# SDK Stable Docs Maintenance

## What the Stable Docs Are

Each package in this repo ships a `docs/` folder. The folder holds four items. These are `OVERVIEW.md`, `DETAILS.md`, `AGENT_MAP.md`, and `topology.svg`. The repo adds `topology.svg` only when the package needs a diagram. `OVERVIEW.md` and `DETAILS.md` teach human readers. `OVERVIEW.md` gives a short orientation. `DETAILS.md` gives a deeper explanation. `AGENT_MAP.md` is a dense index built for AI coding agents. It is exempt from the prose style rules described below.

## Where the Docs Are Authored

These four files are authored directly in this venue repo. This placement is a deliberate exception. Elsewhere, the `SOURCE.md` rule bars hand edits to generated content. Code changes still flow through the canonical source tree. Documentation changes here are welcome as pull requests.

## How the Publisher Preserves the Docs

The publish pipeline wipes `packages/` on each release. It then reloads `packages/` from the canonical tree. Before the wipe, the pipeline stashes the four doc deliverables. After the reload, it restores them. A guard checks the result. The guard fails the publish if a previously present doc file has vanished. Every other file inside a package `docs/` folder comes from the canonical tree. Those other files refresh on each publish.

## How Staleness Is Detected

Every doc file carries front matter. The front matter lists each source file the doc describes. Each listed source carries a git blob hash from authoring time. A blob hash is a git content checksum. The `docs-scrub` tool lives at `scripts/docs-scrub.py` in the canonical repo. The tool recomputes each blob hash. It then reports any drift it finds. A stale hash means one thing. It flags the doc for a true-up pass. A stale hash never triggers an automatic rewrite.

## The True-Up Cadence

Run `docs-scrub` at each release boundary. For each drifted source, judge the kind of change. Comment-only drift needs only a refreshed hash. Code changes need updated prose as well. Update the affected sections of `OVERVIEW.md` or `DETAILS.md`. Refresh the drifted blob hashes in the front matter. Land the finished fix as a docs pull request.

## The Style Gate

Every prose change to `OVERVIEW.md` or `DETAILS.md` must pass `docs-style-lint`. This tool lives at `scripts/docs-style-lint.py` in the canonical repo. The gate enforces four rules. First, no sentence carries more than two commas, unless it lists simple plain items. Second, do not contrast a negated clause with a corrected one in the same sentence. Third, do not deny a claim and then restate it as the correction in one sentence. Fourth, the Flesch-Kincaid reading level must sit at 8.0 or below. One more rule covers every file in `docs/`. An em-dash never appears in any of them. This ban covers `AGENT_MAP.md` and the text inside `topology.svg`.
