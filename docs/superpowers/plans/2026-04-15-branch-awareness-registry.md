# Branch Awareness & Centralized Registry — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add branch safety checks before deploys and a centralized JSON registry so `yeet --status` shows deploy state across all projects from anywhere.

**Architecture:** Two independent features added inline to the existing `yeet` bash script. Branch awareness is a simple guard inserted before the `git ftp` call. The registry is written via `python3 -c` JSON manipulation after each successful deploy, and read back by a `--status` flag handler at the top of the script.

**Tech Stack:** Bash, python3 (for JSON), git-ftp

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

### Task 2: Registry — write deploy data after successful deploy

**Files:**
- Modify: `yeet:229-246` (inside the `FTP_EXIT -eq 0` success block)

- [ ] **Step 1: Add the registry update function**

Insert this function after the `refresh_status()` function (after line 98, before the `# === Preflight ===` comment):

```bash
# === Update global registry ===
update_registry() {
  local project_path env_name
  project_path=$(pwd)
  env_name="$1"

  local registry_dir="$HOME/.config/yeet"
  local registry_file="$registry_dir/registry.json"
  local short_hash date_stamp commit_msg

  short_hash=$(git log --format="%h" -1 HEAD)
  date_stamp=$(date "+%Y-%m-%d %H:%M")
  commit_msg=$(git log --format="%s" -1 HEAD)

  mkdir -p "$registry_dir"

  python3 -c "
import json, os, sys

registry_file = sys.argv[1]
project = sys.argv[2]
env = sys.argv[3]
data = {
    'last_deploy': sys.argv[4],
    'commit_hash': sys.argv[5],
    'commit_message': sys.argv[6]
}

registry = {}
if os.path.exists(registry_file):
    with open(registry_file) as f:
        registry = json.load(f)

registry.setdefault(project, {})[env] = data

with open(registry_file, 'w') as f:
    json.dump(registry, f, indent=2)
" "$registry_file" "$project_path" "$env_name" "$date_stamp" "$short_hash" "$commit_msg"
}
```

- [ ] **Step 2: Call update_registry after successful deploy**

In the success block (inside the `if [ $FTP_EXIT -eq 0 ]` block, after line 236 `[ -n "$URL" ] && echo -e ...`), add:

```bash
  update_registry "$LABEL"
```

Place it before the sound playback, so the registry is updated even if sound fails. The success block should now read:

```bash
if [ $FTP_EXIT -eq 0 ]; then
  echo
  echo -e "  ${GREEN}✓ Deployed to ${BOLD}${LABEL}${NC}"
  [ -n "$URL" ] && echo -e "  ${DIM}→ ${URL}${NC}"

  update_registry "$LABEL"

  # Play completion sound
  SOUND_FILE="$HOME/.config/yeet/sound.mp3"
  if [ -f "$SOUND_FILE" ]; then
    afplay "$SOUND_FILE" &
  else
    printf '\a'
  fi

  refresh_status
fi
```

- [ ] **Step 3: Manual test — verify registry write**

```bash
# Create a test registry entry manually to verify the function works
# Add a temporary call at the end of the script for testing:
# update_registry "TEST"
# Run ./yeet (pick quit immediately after adding the temp call at the bottom)
# Then check:
cat ~/.config/yeet/registry.json
# Should show current project path with a TEST entry containing hash, message, timestamp

# Clean up: remove the temporary call, and remove the TEST entry from registry.json
```

- [ ] **Step 4: Commit**

```bash
git add yeet
git commit -m "added registry update after successful deploy"
```

---

### Task 3: Registry — global status via `yeet --status`

**Files:**
- Modify: `yeet` — add `--status` flag parsing at the top (before interactive menu), add `global_status()` function

- [ ] **Step 1: Add the global_status function**

Insert this function after `update_registry()` (before the `# === Preflight ===` comment):

```bash
# === Global deploy status ===
global_status() {
  local registry_file="$HOME/.config/yeet/registry.json"

  if [ ! -f "$registry_file" ]; then
    echo -e "  ${DIM}No deployments recorded yet.${NC}"
    exit 0
  fi

  python3 -c "
import json, subprocess, os, sys

registry_file = sys.argv[1]
with open(registry_file) as f:
    registry = json.load(f)

if not registry:
    print('  No deployments recorded yet.')
    sys.exit(0)

for project_path in sorted(registry):
    project_name = os.path.basename(project_path)
    print(f'  \033[1m{project_name}\033[0m  \033[2m{project_path}\033[0m')

    envs = registry[project_path]
    for env_name in sorted(envs):
        entry = envs[env_name]
        h = entry.get('commit_hash', '?')
        msg = entry.get('commit_message', '')
        ts = entry.get('last_deploy', '?')

        if len(msg) > 40:
            msg = msg[:37] + '...'

        behind = ''
        if os.path.isdir(project_path):
            try:
                full_hash = subprocess.check_output(
                    ['git', 'log', '--format=%H', '-1', h],
                    cwd=project_path, stderr=subprocess.DEVNULL
                ).decode().strip()
                count = subprocess.check_output(
                    ['git', 'rev-list', '--count', full_hash + '..HEAD'],
                    cwd=project_path, stderr=subprocess.DEVNULL
                ).decode().strip()
                behind = f'({count} behind HEAD)'
            except subprocess.CalledProcessError:
                behind = '(unknown commit)'
        else:
            behind = '(repo not found)'

        print(f'    {env_name:<8} {h}  {ts}  \"{msg}\"  {behind}')

    print()
" "$registry_file"
}
```

- [ ] **Step 2: Add --status flag parsing**

Insert this block after the `.env` loading and `DEPLOY_BRANCH` setup, but before the `# === Select action ===` section. Place it right before the `clear` on line 114:

```bash
# === Global status flag ===
if [ "$1" = "--status" ]; then
  logo
  global_status
  exit 0
fi
```

- [ ] **Step 3: Manual test — verify global status**

```bash
# First, seed the registry with test data:
mkdir -p ~/.config/yeet
cat > ~/.config/yeet/registry.json << 'TESTJSON'
{
  "/Users/felixf/Code/yeet": {
    "DEV": {
      "last_deploy": "2026-04-15 14:30",
      "commit_hash": "8776259",
      "commit_message": "added design spec for branch awareness and centralized registry"
    }
  }
}
TESTJSON

# Run from any directory:
./yeet --status
# Should show the yeet project with DEV env, commit info, and "N behind HEAD"

# Run from a different directory:
cd /tmp && /Users/felixf/Code/yeet/yeet --status
# Should show the same output

# Clean up test data if desired, or leave it — real deploys will overwrite
```

- [ ] **Step 4: Commit**

```bash
git add yeet
git commit -m "added global status via yeet --status"
```

---

### Task 4: Update README

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

- [ ] **Step 2: Add global status docs**

Add this section after "## Branch Awareness":

```markdown
## Global Status

See the deploy status of all your yeet-enabled projects from anywhere:

```
yeet --status
```

This reads from `~/.config/yeet/registry.json`, which is updated automatically after each successful deploy. If a project's git repo is accessible, it also shows how many commits behind HEAD each environment is.
```

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "added docs for branch awareness and global status"
```
