# Branch Awareness — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a branch safety check before deploys so you get a confirmation prompt when deploying from an unexpected branch.

**Architecture:** A simple guard inserted into the existing `yeet` bash script, between environment selection and the `git ftp` call. The expected branch is configurable via `DEPLOY_BRANCH` in `.env`, defaulting to `main`.

**Tech Stack:** Bash, git-ftp

---

### Task 1: Branch awareness — add the deploy branch check

**Files:**
- Modify: `yeet:107-109` (after `.env` is loaded, read `DEPLOY_BRANCH`)
- Modify: `yeet:203-217` (after the confirmation screen, before `git ftp` call, insert branch check)

- [ ] **Step 1: Add DEPLOY_BRANCH variable after .env is loaded**

After line 109 (`source .env`), add the branch resolution. Insert this right after the `.env` loading block (after line 111, before the `# === Select action ===` comment):

```bash
# === Deploy branch ===
DEPLOY_BRANCH="${DEPLOY_BRANCH:-main}"
```

- [ ] **Step 2: Add branch check before deploy**

Insert this block after the password prompt section (after line 222, the closing `fi` of the missing-credentials tip) and before the `git ftp` call on line 224:

```bash
# === Branch check ===
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
if [ "$CURRENT_BRANCH" != "$DEPLOY_BRANCH" ]; then
  echo -e "  ${RED}⚠ You're on '${CURRENT_BRANCH}', not '${DEPLOY_BRANCH}'.${NC}"
  read -p "  Deploy anyway? [y/N]: " CONFIRM_BRANCH
  if [[ ! "$CONFIRM_BRANCH" =~ ^[Yy]$ ]]; then
    echo -e "  ${DIM}Aborted.${NC}"
    exit 0
  fi
  echo
fi
```

- [ ] **Step 3: Manual test — verify branch check works**

Run these from the yeet project directory:

```bash
# Test 1: On main, should NOT show warning
git checkout main
./yeet
# Pick 1) Push, then any env — should go straight to deploy (will fail without FTP, that's fine)

# Test 2: On a test branch, SHOULD show warning
git checkout -b test/branch-check
./yeet
# Pick 1) Push, then any env — should show "You're on 'test/branch-check', not 'main'"
# Press enter (default N) — should abort
# Run again, type 'y' — should proceed to deploy

# Clean up
git checkout main
git branch -d test/branch-check
```

- [ ] **Step 4: Commit**

```bash
git add yeet
git commit -m "added branch awareness check before deploy"
```

---

### Task 2: Update README

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add branch awareness docs**

Add this section after the "## .env configuration" section, before "## Completion Sound":

```markdown
## Branch Awareness

By default, yeet warns you when deploying from a branch other than `main`. You'll see a confirmation prompt before proceeding.

To change the expected branch, add to your `.env`:

```env
DEPLOY_BRANCH=develop
```
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "added docs for branch awareness"
```
