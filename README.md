# DMV Scrapers Public Runner

This is a **public GitHub repo** whose sole purpose is to run scheduled scraping workflows for private DMV scraper repos. Because this repo is public, GitHub Actions minutes are **unlimited and free**.

The actual scraper code lives in separate private repos. This repo checks them out at workflow runtime, executes the scraper, and commits results back.

## Architecture

```
Private repos (code stays private)       Public runner (this repo)
────────────────────────────────         ────────────────────────────
  md-dmv-scraper/                          .github/workflows/
    waittime.py                              _scraper-template.yml  ← reusable logic
    visittime_services.py                    md-daily.yml           ← MD daily schedule
    ...                                      md-monthly.yml         ← MD monthly schedule
                                             va.yml                 ← VA schedule
  va-dmv-scraper/                            oh.yml                 ← OH schedule
    va_dmv_scraper.py
    ...

  oh-dmv-scraper/
    ohio_wait_times_scraper.py
    ...
```

## One-time setup

### Step 1: Create a Personal Access Token (PAT)

This token lets the public runner check out and push to your private repos.

1. Go to https://github.com/settings/tokens
2. Click **Generate new token** → **Generate new token (classic)**
3. Name it something like `dmv-scraper-runner`
4. Set expiration to whatever you want (e.g. 1 year — you'll need to rotate it then)
5. Select the following scope:
   - ✅ **repo** (full control of private repositories — needed for checkout AND push)
6. Click **Generate token** and **copy the token immediately** (you won't see it again)

> **Note on fine-grained tokens:** You can use a fine-grained PAT instead if you prefer. Grant it **Contents: Read and write** access to only the `md-dmv-scraper`, `va-dmv-scraper`, and `oh-dmv-scraper` repos. This is more secure but slightly more setup.

### Step 2: Add the token as a secret in this repo

1. In this repo, go to **Settings → Secrets and variables → Actions**
2. Click **New repository secret**
3. Name: `PRIVATE_REPO_TOKEN`
4. Value: paste the PAT from Step 1
5. Click **Add secret**

### Step 3: Add the VA-specific DMV secrets

The VA scraper requires additional cookies and headers. Add these as repo secrets too:

1. **Settings → Secrets and variables → Actions → New repository secret**
2. Add `DMV_COOKIES` with the value from your current VA private repo's secrets
3. Add `DMV_HEADERS` with the value from your current VA private repo's secrets

> You can view your existing secrets' **names** in the private repo settings but not their values. You'll need to re-enter the values here. If you don't have them saved, you'll need to regenerate them.

### Step 4: Update `OWNER` in each workflow file

Each caller workflow has a `target_repo` line like:
```yaml
target_repo: OWNER/md-dmv-scraper
```

Replace `OWNER` with your actual GitHub username (or org name) in all four files:
- `.github/workflows/md-daily.yml`
- `.github/workflows/md-monthly.yml`
- `.github/workflows/va.yml`
- `.github/workflows/oh.yml`

### Step 5: Disable the old workflows in the private repos

To avoid both runners firing at once:

1. In each private repo, go to **Actions** tab
2. For each existing workflow, click the workflow name → **⋯ menu → Disable workflow**

Alternatively, you can delete or rename the old `.yml` files in the private repos.

### Step 6: Test manually

Each workflow has `workflow_dispatch` enabled, so you can trigger a test run without waiting for the schedule:

1. Go to **Actions** tab in this repo
2. Click on one of the workflows (e.g. "OH DMV Scraper")
3. Click **Run workflow** → **Run workflow**
4. Watch the run complete and verify that data was committed to the private repo

## Adding more scrapers later

To add another scraper, create a new file in `.github/workflows/` based on one of the existing callers. The pattern is:

```yaml
name: XX DMV Scraper

on:
  schedule:
    - cron: '...'   # your schedule
  workflow_dispatch:

jobs:
  call-scraper:
    uses: ./.github/workflows/_scraper-template.yml
    with:
      target_repo: OWNER/xx-dmv-scraper
      python_version: '3.11'
      requirements_file: 'requirements.txt'   # OR use pip_packages instead
      run_commands: |
        python your_scraper.py --once
      commit_paths: '*.csv'
    secrets:
      PRIVATE_REPO_TOKEN: ${{ secrets.PRIVATE_REPO_TOKEN }}
```

## Security considerations

- ✅ Private repo code is never copied into this public repo — it's checked out at runtime and discarded
- ✅ Secrets are stored in this repo's GitHub Secrets and are never exposed in logs
- ⚠️ **Workflow run logs are public** — anyone can see them. Make sure your scraper scripts don't `echo` or `print` sensitive values (API keys, cookies, etc.)
- ⚠️ The PAT has access to all your private repos. If it leaks, rotate immediately at https://github.com/settings/tokens

## Files in this repo

| File | Purpose |
|------|---------|
| `.github/workflows/_scraper-template.yml` | Reusable workflow (shared execution logic) |
| `.github/workflows/md-daily.yml` | MD daily scraper (3x/weekday) |
| `.github/workflows/md-monthly.yml` | MD monthly scraper (1st & 29th) |
| `.github/workflows/va.yml` | VA scraper |
| `.github/workflows/oh.yml` | OH scraper |
| `README.md` | This file |
