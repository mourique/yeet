# Branch Awareness & Centralized Registry

Two features for yeet: warn when deploying from the wrong branch, and track all deployments in a central registry.

## Branch Awareness

### Behavior

- Before deploying (`push` or `init`), check the current git branch against the configured deploy branch
- If the branch doesn't match, prompt: `"You're on 'feature/xyz', not 'main'. Deploy anyway? [y/N]"`
- Default answer is No — pressing enter aborts the deploy
- The check runs after environment selection, before the deploy execution

### Configuration

- `DEPLOY_BRANCH` in `.env` sets the expected branch (e.g. `DEPLOY_BRANCH=main`)
- Falls back to `main` if not set
- Global setting — applies to all environments equally

## Centralized Project Registry

### Storage

- File: `~/.config/yeet/registry.json`
- Structure:

```json
{
  "/Users/felixf/Code/my-site": {
    "PROD": {
      "last_deploy": "2026-04-15 14:30",
      "commit_hash": "a1b2c3d",
      "commit_message": "fix navbar spacing"
    },
    "DEV": {
      "last_deploy": "2026-04-14 09:12",
      "commit_hash": "f4e5d6c",
      "commit_message": "add contact form validation"
    }
  }
}
```

- Keyed by absolute project path, then by environment name

### Write behavior

- After every successful deploy (`git ftp` exit code 0), upsert the registry entry for the current project path and environment
- Store: `last_deploy` (timestamp), `commit_hash` (short hash from HEAD), `commit_message` (commit subject line)
- Create `~/.config/yeet/` directory if it doesn't exist
- Create `registry.json` if it doesn't exist (start with `{}`)
- Use `python3 -c` for JSON read/write (available on macOS by default, avoids `jq` dependency)

### Global status: `yeet --status`

- Works from any directory — no need to be inside a project
- Reads `registry.json` and iterates over all projects and environments
- For each entry:
  - If the project path exists on disk: `cd` into it, run `git rev-list --count <hash>..HEAD` to calculate commits behind, display enriched status
  - If the project path doesn't exist: show cached info with a "(repo not found)" note
  - If the commit hash is unknown to the local repo (e.g. after a rebase): show "(unknown commit)" note
- Output format: table similar to existing per-project status, grouped by project path

### Existing behavior unchanged

- Per-project `.deploy-status` file continues to be written as before
- The registry is purely additive — removing it doesn't break anything

## Implementation approach

- All changes go into the single `yeet` script (Approach A)
- Parse `--status` flag before the interactive menu; if present, show global status and exit
- Branch check inserted after environment selection, before deploy execution
- JSON manipulation via `python3 -c` one-liners
