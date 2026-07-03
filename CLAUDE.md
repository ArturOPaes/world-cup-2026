# WC26 Feature Development Harness

This directory is an AI engineering exercise harness. It does not contain app code.

## Exercise Pillars
1. Read task from GitHub Issues (via `gh` CLI)
2. Multi-agent coordination (PO → Developer → QA → Deploy)
3. One skill: `/wc26-feature <issue-number>`
4. Human gates: two AskUserQuestion approval checkpoints
5. One eval: QA agent scores 0-10; score ≥ 7 is the automated gate

## Target App
- Repo: `ArturOPaes/world-cup-2026`
- Single file: `index.html` (871 lines, pure HTML/CSS/JS, no build step)
- Hosting: GitHub Pages at `https://arturpaes.github.io/world-cup-2026/`
- Data: fetches from openfootball API at runtime (auto-polls every 120s)

## How to Use
```bash
# Run the full pipeline for a GitHub Issue
/wc26-feature <issue-number>
```

The skill reads a GitHub Issue, coordinates 4 AI agents through the full
development cycle, and deploys to GitHub Pages with two human approval checkpoints.

## Agent Roles
| Agent | File | Responsibility |
|-------|------|---------------|
| PO | `agents/po-agent.md` | Reads issue → structured plan + GitHub Project item |
| Developer | `agents/developer-agent.md` | Creates branch, edits `index.html`, pushes |
| QA | `agents/qa-agent.md` | Diffs branch, scores 0-10 on 4 dimensions |
| Deploy | `agents/deploy-agent.md` | PR → merge → GitHub Pages → verifies HTTP 200 |

## Runtime State
Each pipeline run uses `/tmp/wc26-<issue-number>/` for inter-agent state:
- `po-plan.md` — PO's structured implementation plan
- `project-item.txt` — GitHub Project item URL
- `branch.txt` — feature branch name
- `dev-summary.txt` — developer's implementation notes
- `qa-report.md` — QA scored report (contains `SCORE: N/10` line)
- `deploy-result.txt` — PR URL, Pages status, HTTP status

## GitHub Setup
- App repo: `ArturOPaes/world-cup-2026`
- `gh` CLI must be authenticated as `ArturOPaes`
- Required token scopes: `repo`, `project`

## index.html Anatomy (for agents)
```
Lines   1-180:  CSS (custom properties, layout, animations)
Lines 181-200:  HTML structure (header, .stage div, footer)
Line    ~215:   JS CONFIG (FEED_URL, STADIUM map, NATION map, ISO map)
Lines ~290-390: JS DATA (bracket skeleton: CHILDREN, RADIUS, ANGLE)
Lines ~425-650: JS RENDER (SVG lines, flag placement, match status logic)
Lines ~650-800: JS INTERACTION (tooltips, venue toggle, event listeners)
Lines ~827-869: JS LOAD (fetchResults, normalize, buildModel, load, stamp)
```
