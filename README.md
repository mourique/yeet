# yeet — FTP Deploy Script

Interactive deploy script for web projects using [git-ftp](https://github.com/git-ftp/git-ftp). Choose an action (push, init, status) and a target environment. Environments are auto-detected from `DEPLOY_*` vars in your project's `.env`.

## Setup

1. Install yeet globally:
   ```
   cp yeet ~/.local/bin/yeet
   ```
2. Create a `.env` in your project root with your deploy credentials (see below)
3. Create a `.git-ftp-ignore` in your project root (see below)
4. Run `yeet` from your project directory

## .git-ftp-ignore

Place this file in your project root to exclude files from FTP deployment:

```
content/
media/
.env
.deploy-status
```

## .git-ftp-include

If you use Composer (e.g. for Kirby CMS), `vendor/` and `kirby/` are typically git-ignored but need to be deployed. Create a `.git-ftp-include` in your project root:

```
kirby/:composer.lock
vendor/:composer.lock
```

This tells git-ftp to upload these directories whenever `composer.lock` changes.

## Finding your HOST path

1. Connect via FTP to see your root directory:
   ```
   curl --user ftp-user ftp://yourserver.com/
   ```
2. If you see your project files, you're already in the project root:
   ```
   HOST=ftp://yourserver.com
   ```
3. If your project is in a subfolder (e.g. `public_html/`), add it to the path:
   ```
   HOST=ftp://yourserver.com/public_html
   ```
4. You can browse deeper with:
   ```
   curl --user ftp-user ftp://yourserver.com/public_html/
   ```

## .env configuration

Add as many environments as you need. `DEPLOY_*_URL` is optional — it stores the public URL where the environment is accessible and is shown during deploy and in `.deploy-status`.

```env
DEPLOY_DEV_HOST=ftp://dev.yourserver.com
DEPLOY_DEV_USER=ftp-dev-user
DEPLOY_DEV_PASS=your-dev-password
DEPLOY_DEV_URL=https://dev.yourserver.com

DEPLOY_PROD_HOST=ftp://yourserver.com
DEPLOY_PROD_USER=ftp-prod-user
DEPLOY_PROD_PASS=your-prod-password
DEPLOY_PROD_URL=https://yourserver.com

DEPLOY_STAGING_HOST=ftp://staging.yourserver.com
DEPLOY_STAGING_USER=ftp-staging-user
DEPLOY_STAGING_PASS=your-staging-password
DEPLOY_STAGING_URL=https://staging.yourserver.com
```

## Requirements

- [git-ftp](https://github.com/git-ftp/git-ftp) (`brew install git-ftp`)
