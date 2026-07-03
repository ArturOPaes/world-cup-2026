# World Cup 2026 Bracket

Live interactive bracket for the 2026 FIFA World Cup, hosted on GitHub Pages:
**https://arturpaes.github.io/world-cup-2026/**

A single self-contained `index.html` (HTML/CSS/JS, no build step, no external
libraries) that fetches match results from the openfootball API at runtime
and auto-polls every 120 seconds.

## How this project is developed

Every change to `index.html` in this repo goes through an automated,
multi-agent pipeline (defined in a companion harness project) rather than
being hand-edited directly. The pipeline reads a GitHub Issue and drives it
through four AI agent roles with two human approval checkpoints before
anything reaches `main`.

### Command

```
/wc26-feature <issue-number>
```

Run against a GitHub Issue number on this repo. The issue describes the
feature or fix; the pipeline does the rest.

### Pipeline flow

```
GitHub Issue
     |
     v
[PO Agent]        analyzes the issue, writes a structured implementation
                   plan (acceptance criteria, exact index.html sections/line
                   ranges to touch, QA checklist), files it on the project board
     |
     v
 >> GATE 1 (human) <<   approve the plan, or reject and edit the issue
     |
     v
[Developer Agent] creates a feature branch, implements the plan in
                   index.html, commits, pushes
     |
     v
[QA Agent]         diffs the branch against main, scores it 0-10 across
                   4 weighted dimensions:
                     - Issue Alignment   (max 3)
                     - Code Quality      (max 3)
                     - Minimal Footprint (max 2)
                     - No Regressions    (max 2)
     |
     v
 >> EVAL GATE (automated) <<   score >= 7 = pass; score < 7 surfaces a
                                warning at Gate 2 but does not hard-block —
                                a human can still override
     |
     v
 >> GATE 2 (human) <<   approve merge + deploy, or reject and leave the
                         branch open for iteration
     |
     v
[Deploy Agent]     opens a PR, squash-merges to main, deletes the branch,
                   polls GitHub Pages until the build is "built", verifies
                   the live site returns HTTP 200, closes the issue with a
                   link to the PR
```

### Why two human gates + one automated eval

- **Gate 1** exists because a bad plan produces a bad implementation no
  matter how good the developer step is — cheaper to catch scope/interpretation
  issues before any code is written.
- **The QA score** is the automated quality bar: it forces the diff to
  actually satisfy the issue's acceptance criteria, follow the codebase's
  existing conventions (CSS custom properties only, `escapeHTML`/`escapeAttr`
  for anything dynamic, no hardcoded colors/results), stay minimal, and not
  regress the fetch → normalize → buildModel → render → load pipeline.
- **Gate 2** exists because a passing score is a floor, not a substitute for
  judgment — a human decides whether a 10/10 diff is actually worth shipping
  right now, and can still choose to ship (or not ship) a sub-7 diff with
  full awareness of why it scored low.

### Known operational quirk: GitHub Pages deploy concurrency

GitHub Pages (legacy build type, deploying from the `main` branch root)
processes one deployment at a time per repo. Merging several PRs back-to-back
in a short window can produce `Error: Deployment failed, try again later` on
the in-between runs, and the Pages status can briefly show `errored`. This is
not a hard quota — it self-resolves on the next push once the queue clears.
If Pages looks stuck, check `gh run list` for a run wedged in `queued` and
give it a few minutes before assuming something is actually broken.

## Repo layout

- `index.html` — the entire app (CSS, markup, and JS in one file)
- `CLAUDE.md`, `agents/`, `.claude/` — the AI engineering harness that
  develops this app (PO/Developer/QA/Deploy agent definitions and the
  `/wc26-feature` skill); not part of the runtime app itself
