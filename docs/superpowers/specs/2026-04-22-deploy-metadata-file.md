# Deploy Metadata File (.deploy.json)

Yeet writes a `.deploy.json` file after each deploy so that beam (a Kirby plugin) can read it and forward deploy information to helm (a centralized dashboard).

## Context

- **yeet** deploys projects via git-ftp to FTP servers
- **beam** is a Kirby plugin that reports system/plugin/content info to helm via HMAC-signed HTTP
- **helm** is a centralized PHP dashboard showing all connected sites
- beam already has a `/beam/status` endpoint that collects system info — deploy data will be added there
- beam currently reads `.git-ftp.log` for the deploy commit hash — this will be replaced by `.deploy.json`

## .deploy.json

### Location

- Project root (same level as `.git-ftp.log`)
- Git-ignored (added to `.gitignore`)
- Uploaded to the remote host by yeet via FTP after each deploy

### Contents

```json
{
  "commit": "a1b2c3d",
  "commitFull": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0",
  "message": "fix navbar spacing",
  "branch": "main",
  "environment": "PROD",
  "timestamp": "2026-04-22T14:30:00+02:00"
}
```

### Field Definitions

| Field         | Type   | Description                                      |
|---------------|--------|--------------------------------------------------|
| `commit`      | string | Short SHA (7 chars)                              |
| `commitFull`  | string | Full 40-char SHA                                 |
| `message`     | string | First line of the commit message                 |
| `branch`      | string | Git branch that was deployed from                |
| `environment` | string | Target environment name (DEV, PROD, STAGING, etc.) |
| `timestamp`   | string | ISO 8601 deploy timestamp with timezone offset   |

### Behavior

- Overwritten on every successful deploy (push or init) — contains only the latest deploy
- Written locally by yeet, then uploaded via git-ftp alongside the deployed files
- If the deploy fails, the file is not written/updated

## Changes to yeet

### After a successful deploy:

1. Collect deploy metadata (commit, message, branch, environment, timestamp)
2. Write `.deploy.json` to the project root
3. The file is included in the git-ftp upload because it exists in the working directory

### Considerations

- `.deploy.json` must NOT be in `.git-ftp-ignore` (so it gets uploaded)
- `.deploy.json` SHOULD be in `.gitignore` (it's generated, not source code)
- yeet should create/overwrite this file before the git-ftp push/init command runs, so it's included in the upload
- On `init`, the file is part of the initial full upload
- On `push`, since git-ftp tracks changes via git, and `.deploy.json` is git-ignored, yeet may need to handle this via a separate FTP upload step after the git-ftp push completes

### git-ftp and git-ignored files

Since `.deploy.json` is in `.gitignore`, git-ftp will not upload it automatically (git-ftp only uploads git-tracked changes). Two options:

1. **Use `.git-ftp-include`** — add `.deploy.json` to the project's `.git-ftp-include` file so git-ftp always uploads it
2. **Upload separately via curl/FTP** — yeet uploads `.deploy.json` directly via FTP after the git-ftp command completes

Option 1 is simpler and uses existing git-ftp mechanics. Recommended approach: yeet documents that `.deploy.json` should be listed in `.git-ftp-include`.

## Changes to beam

### Status.php

- Remove the `.git-ftp.log` reading logic
- Read `.deploy.json` from the Kirby project root instead
- Parse the JSON and include it in the status response under a `deploy` key

### Updated status response structure

```json
{
  "site": { "id": "...", "label": "...", "url": "..." },
  "system": {
    "kirbyVersion": "5.0.0",
    "phpVersion": "8.3.0",
    "serverTime": "2026-04-22T14:35:00+02:00"
  },
  "deploy": {
    "commit": "a1b2c3d",
    "commitFull": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0",
    "message": "fix navbar spacing",
    "branch": "main",
    "environment": "PROD",
    "timestamp": "2026-04-22T14:30:00+02:00"
  },
  "content": { "pages": 42, "files": 128, "bytes": 5242880, "generated": "..." },
  "plugins": [ "..." ]
}
```

- If `.deploy.json` does not exist or is unreadable, `deploy` should be `null`
- The `deployCommit` field is removed from `system` (replaced by `deploy.commit`)

## Changes to helm

### Table view

- Add a "Last Deploy" column showing the deploy timestamp (human-readable relative time, e.g., "2 hours ago")
- If `deploy` is `null`, show "—" or "Unknown"

### Detail modal

Display all deploy fields:

- **Commit:** short hash (link to git remote if available, plain text otherwise)
- **Message:** full commit message
- **Branch:** branch name
- **Environment:** environment label
- **Deployed:** formatted timestamp (absolute + relative)

## Affected Repositories

| Repo | Changes |
|------|---------|
| yeet | Generate `.deploy.json` after successful deploy |
| beam | Read `.deploy.json` instead of `.git-ftp.log` in Status.php |
| helm | Display deploy info in table and modal |

## Out of Scope

- Deploy history (only latest deploy is stored)
- yeet calling helm directly
- Backward compatibility with `.git-ftp.log` reading in beam
