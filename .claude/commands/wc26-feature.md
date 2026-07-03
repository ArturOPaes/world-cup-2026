---
description: Full dev pipeline for WC26 — reads a GitHub Issue and runs PO → Developer → QA → Deploy with human approval gates
argument-hint: <issue-number>
allowed-tools: Task, AskUserQuestion, Read, Write, Bash(gh:*), Bash(git:*), Bash(curl:*), Bash(mkdir:*), Bash(cat:*), Bash(grep:*), Bash(python3:*), Bash(rm:*), Bash(echo:*), Bash(sleep:*), Bash(ls:*)
model: sonnet
---

# WC26 Feature Pipeline

You are the orchestrator for the WC26 multi-agent development pipeline.
Process GitHub Issue #$ARGUMENTS from `ArturOPaes/world-cup-2026` through
the full cycle: PO → Developer → QA → Deploy.

The pipeline uses `/tmp/wc26-$ARGUMENTS/` as the shared state directory.
All agents read and write files there.

---

## Step 0: Setup and bootstrap check

Create the state directory:
```bash
mkdir -p /tmp/wc26-$ARGUMENTS
```

Verify the target repo exists and is accessible:
```bash
gh repo view ArturOPaes/world-cup-2026 --json name,url 2>&1
```

If the command fails (repo not found or auth error), stop and tell the user:
> "Cannot access ArturOPaes/world-cup-2026. Make sure you are authenticated as ArturOPaes:
> run `gh auth login` then `gh auth switch --user ArturOPaes`"

Check if index.html exists in the repo:
```bash
gh api repos/ArturOPaes/world-cup-2026/contents/index.html --jq '.name' 2>/dev/null || echo "MISSING"
```

If index.html is MISSING, initialize the app content from ShadyMccoy/WC26:
```bash
gh repo clone ArturOPaes/world-cup-2026 ~/wc26-workspace 2>/dev/null || (git -C ~/wc26-workspace fetch origin && git -C ~/wc26-workspace checkout main && git -C ~/wc26-workspace pull)
curl -s https://raw.githubusercontent.com/ShadyMccoy/WC26/main/index.html > ~/wc26-workspace/index.html
git -C ~/wc26-workspace add index.html
git -C ~/wc26-workspace commit -m "feat: init WC26 bracket from ShadyMccoy/WC26"
git -C ~/wc26-workspace push origin main
```

Also check if GitHub Pages is enabled. If not, enable it:
```bash
PAGES_STATUS=$(gh api repos/ArturOPaes/world-cup-2026/pages --jq '.status' 2>/dev/null || echo "MISSING")
if [ "$PAGES_STATUS" = "MISSING" ]; then
  gh api repos/ArturOPaes/world-cup-2026/pages \
    -X POST \
    -H "Accept: application/vnd.github+json" \
    -f "source[branch]=main" \
    -f "source[path]=/"
fi
```

---

## Step 1: Read the GitHub Issue

Fetch the issue details:
```bash
gh issue view $ARGUMENTS \
  --repo ArturOPaes/world-cup-2026 \
  --json number,title,body,url
```

If the issue does not exist, stop and tell the user:
> "Issue #$ARGUMENTS not found in ArturOPaes/world-cup-2026. Create it first with:
> `gh issue create --repo ArturOPaes/world-cup-2026 --title '...' --body '...'`"

Extract and store: ISSUE_TITLE, ISSUE_BODY, ISSUE_URL for use in subsequent steps.

Tell the user:
> "Reading issue #$ARGUMENTS: [ISSUE_TITLE]"

---

## Step 2: PO Agent — analyze and plan

Tell the user: "Running PO Agent..."

Launch a Task subagent with the contents of `./agents/po-agent.md` as the system prompt.
Pass it a user message with:
```
ISSUE_NUM: $ARGUMENTS
ISSUE_TITLE: [extracted title]
ISSUE_BODY: [extracted body]
ISSUE_URL: [extracted URL]
STATE_DIR: /tmp/wc26-$ARGUMENTS
```

The PO agent will write `/tmp/wc26-$ARGUMENTS/po-plan.md` before it exits.

After the Task completes, read the plan:
```bash
cat /tmp/wc26-$ARGUMENTS/po-plan.md
```

If the file does not exist, tell the user the PO agent failed to write its output and stop.

---

## Gate 1: Human approval of PO plan

Display the full contents of `/tmp/wc26-$ARGUMENTS/po-plan.md` to the user.

Use AskUserQuestion with:
- header: "Approve plan"
- question: "The PO Agent has produced the plan above for issue #$ARGUMENTS. Do you approve proceeding with development?"
- options:
  - label: "Approve — proceed to development"
    description: "The Developer Agent will implement this plan on a feature branch"
  - label: "Reject — stop pipeline"
    description: "The pipeline stops here. Edit the issue and re-run /wc26-feature $ARGUMENTS"

If the user selects Reject, clean up and stop:
```bash
rm -rf /tmp/wc26-$ARGUMENTS
```
Tell the user: "Pipeline stopped at Gate 1. Edit issue #$ARGUMENTS and re-run `/wc26-feature $ARGUMENTS` when ready."

---

## Step 3: Developer Agent — implement on a feature branch

Tell the user: "Running Developer Agent..."

Read the PO plan from `/tmp/wc26-$ARGUMENTS/po-plan.md`.

Launch a Task subagent with the contents of `./agents/developer-agent.md` as the system prompt.
Pass it a user message with:
```
ISSUE_NUM: $ARGUMENTS
ISSUE_TITLE: [extracted title]
PO_PLAN:
[full contents of /tmp/wc26-$ARGUMENTS/po-plan.md]
STATE_DIR: /tmp/wc26-$ARGUMENTS
```

After the Task completes, read the branch name:
```bash
cat /tmp/wc26-$ARGUMENTS/branch.txt
```

Store the branch name as BRANCH_NAME. If branch.txt is missing, tell the user the Developer Agent failed and stop.

Tell the user: "Developer Agent pushed branch: [BRANCH_NAME]"

---

## Step 4: QA Agent — review diff and score

Tell the user: "Running QA Agent..."

Read the PO plan for context.

Launch a Task subagent with the contents of `./agents/qa-agent.md` as the system prompt.
Pass it a user message with:
```
ISSUE_NUM: $ARGUMENTS
BRANCH_NAME: [BRANCH_NAME from branch.txt]
PO_PLAN:
[full contents of /tmp/wc26-$ARGUMENTS/po-plan.md]
STATE_DIR: /tmp/wc26-$ARGUMENTS
```

After the Task completes, read the QA report:
```bash
cat /tmp/wc26-$ARGUMENTS/qa-report.md
```

---

## Eval gate: extract and check score

Extract the numeric score from the QA report:
```bash
grep "^SCORE:" /tmp/wc26-$ARGUMENTS/qa-report.md | python3 -c "
import sys
line = sys.stdin.read().strip()
score = int(line.split(':')[1].strip().split('/')[0].strip())
print(score)
"
```

Store as QA_SCORE.

If QA_SCORE < 7, the eval gate FAILS. Note this — you will add a warning to Gate 2.

---

## Gate 2: Human approval of QA report and merge

Display the full contents of `/tmp/wc26-$ARGUMENTS/qa-report.md` to the user.

Build the gate question dynamically:
- If QA_SCORE >= 7: question = "QA Agent scored [QA_SCORE]/10 (PASS). Approve merging to main and deploying to GitHub Pages?"
- If QA_SCORE < 7: question = "⚠️ QA Agent scored [QA_SCORE]/10 (BELOW THRESHOLD of 7). The eval gate failed automatically. You can still approve or reject."

Use AskUserQuestion with:
- header: "Approve merge"
- question: [dynamic question above]
- options:
  - label: "Approve — merge and deploy"
    description: "Creates PR, merges to main, verifies GitHub Pages"
  - label: "Reject — stop pipeline"
    description: "The pipeline stops here. The feature branch remains for iteration."

If the user selects Reject, tell the user:
"Pipeline stopped at Gate 2. Feature branch [BRANCH_NAME] remains at ArturOPaes/world-cup-2026 for revision."
Do not clean up `/tmp/wc26-$ARGUMENTS/` so the user can reference the QA report.

---

## Step 5: Deploy Agent — PR, merge, verify

Tell the user: "Running Deploy Agent..."

Launch a Task subagent with the contents of `./agents/deploy-agent.md` as the system prompt.
Pass it a user message with:
```
ISSUE_NUM: $ARGUMENTS
BRANCH_NAME: [BRANCH_NAME]
ISSUE_URL: [ISSUE_URL]
STATE_DIR: /tmp/wc26-$ARGUMENTS
```

After the Task completes, read the deploy result:
```bash
cat /tmp/wc26-$ARGUMENTS/deploy-result.txt
```

---

## Step 6: Final summary and cleanup

Extract values from `deploy-result.txt` and present a summary table to the user:

```
Pipeline Complete ✓

Issue:         #$ARGUMENTS — [ISSUE_TITLE]
PO plan:       Approved at Gate 1
Feature branch: [BRANCH_NAME]
QA score:      [QA_SCORE]/10
PR:            [PR_URL from deploy-result.txt]
Pages status:  [PAGES_STATUS from deploy-result.txt]
Live at:       https://arturpaes.github.io/world-cup-2026/
```

Clean up state:
```bash
rm -rf /tmp/wc26-$ARGUMENTS
```
