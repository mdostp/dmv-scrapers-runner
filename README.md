# DMV Scrapers Public Runner

This is a **public GitHub repo** whose sole purpose is to run scheduled scraping workflows for private DMV scraper repos. Because this repo is public, GitHub Actions minutes are **unlimited and free**.

The actual scraper code lives in separate private repos. This repo checks them out at workflow runtime, executes the scraper, and commits results back.

## Architecture

```
Private repos (code stays private)       Public runner (this repo)
────────────────────────────────         ────────────────────────────
  md-dmv-scraper/                          .github/workflows/
  va-dmv-scraper/                            _scraper-template.yml  ← reusable logic
  oh-dmv-scraper/                            md-daily.yml
  nc-dmv-scraper/                            md-monthly.yml
  de-dmv-scraper/                            va.yml
  tn-dmv-scraper/                            oh.yml
  ca-dmv-scraper/                            nc.yml
  wi-dmv-scraper/                            de.yml
  in-dmv-scraper/                            tn.yml
  nc-dmv-appt-avail-scraper/                 ca.yml
  tx-dlo-scraper/                            wi.yml
  ms-apptavail-scraper/                      in.yml
  nj-apptavail-scraper/                      nc-appt.yml
                                             tx.yml
                                             ms.yml
                                             nj.yml
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

### Step 2: Add the token as a secret in this repo

1. In this repo, go to **Settings → Secrets and variables → Actions**
2. Click **New repository secret**
3. Name: `PRIVATE_REPO_TOKEN`
4. Value: paste the PAT from Step 1
5. Click **Add secret**

### Step 3: Add the VA-specific DMV secrets

The VA scraper requires additional cookies and headers. Add these as repo secrets too:

1. **Settings → Secrets and variables → Actions → New repository secret**
2. Add `DMV_COOKIES` — raw cookie string, semicolon-separated
3. Add `DMV_HEADERS` — JSON object with request headers

See earlier conversation for the exact values captured from DevTools.

### Step 4: Disable the old workflows in the private repos

To avoid both runners firing at once:

1. In each private repo, go to **Actions** tab
2. For each existing workflow, click the workflow name → **⋯ menu → Disable workflow**

Alternatively, you can delete or rename the old `.yml` files in the private repos.

### Step 5: Test manually

Each workflow has `workflow_dispatch` enabled, so you can trigger a test run without waiting for the schedule:

1. Go to **Actions** tab in this repo
2. Click on one of the workflows (e.g. "OH DMV Scraper")
3. Click **Run workflow** → **Run workflow**
4. Watch the run complete and verify that data was committed to the private repo

## Workflows in this repo

| Workflow file | Private repo | Scraper | Python | Notes |
|---|---|---|---|---|
| `md-daily.yml` | `mdostp/md-dmv-scraper` | `waittime.py`, `visittime_services.py` | 3.11 | 3x/weekday |
| `md-monthly.yml` | `mdostp/md-dmv-scraper` | `visittime_30days.py` | 3.11 | 1st & 29th of month |
| `va.yml` | `mdostp/va-dmv-scraper` | `va_dmv_scraper.py` | 3.11 | Needs `DMV_COOKIES` + `DMV_HEADERS` |
| `oh.yml` | `mdostp/oh-dmv-scraper` | `ohio_wait_times_scraper.py` | 3.11 | |
| `nc.yml` | `mdostp/nc-dmv-scraper` | `scrape_dmv.py` | 3.11 | |
| `de.yml` | `mdostp/de-dmv-scraper` | `delaware_waittimes.py` | 3.11 | |
| `tn.yml` | `mdostp/tn-dmv-scraper` | `tn_dmv_scraper.py` | 3.11 | |
| `ca.yml` | `mdostp/ca-dmv-scraper` | `CA_waittime_scraper.py` | 3.11 | |
| `wi.yml` | `mdostp/wi-dmv-scraper` | `WI_waittime_scraper.py` | 3.11 | Playwright |
| `in.yml` | `mdostp/in-dmv-scraper` | `bmv_wait_times.py` | 3.11 | |
| `nc-appt.yml` | `mdostp/nc-dmv-appt-avail-scraper` | `NC_appointments.py` | 3.12 | Selenium |
| `tx.yml` | `mdostp/tx-dlo-scraper` | `texas_dps_waittimes_daily.py` | 3.11 | Playwright |
| `ms.yml` | `mdostp/ms-apptavail-scraper` | 2 scripts (DL + ID) | 3.12 | |
| `nj.yml` | `mdostp/nj-apptavail-scraper` | `scrape_mvc_appointments.py` | 3.12 | |

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
      target_repo: mdostp/xx-dmv-scraper
      python_version: '3.11'
      # Option A: use a requirements.txt inside the private repo
      requirements_file: 'requirements.txt'
      # Option B: or use inline packages
      # pip_packages: 'requests pandas'
      run_commands: |
        python your_scraper.py --once
      commit_paths: '*.csv'
      # Optional: if the scraper uses Playwright
      # needs_playwright: true
      # playwright_version: '1.58.0'
      # Optional: if the scraper needs DMV_COOKIES / DMV_HEADERS
      # needs_dmv_secrets: true
    secrets:
      PRIVATE_REPO_TOKEN: ${{ secrets.PRIVATE_REPO_TOKEN }}
      # DMV_COOKIES: ${{ secrets.DMV_COOKIES }}
      # DMV_HEADERS: ${{ secrets.DMV_HEADERS }}
```

## Reusable template inputs

The `_scraper-template.yml` workflow accepts these inputs:

| Input | Required | Default | Description |
|---|---|---|---|
| `target_repo` | ✅ | — | Private repo to check out, e.g. `mdostp/foo-scraper` |
| `python_version` | — | `'3.11'` | Python version to set up |
| `requirements_file` | — | `''` | Path to requirements.txt inside the target repo |
| `pip_packages` | — | `''` | Space-separated pip packages (used if no requirements_file) |
| `needs_playwright` | — | `false` | Install Playwright Chromium browser |
| `playwright_version` | — | `'1.58.0'` | Playwright version (for cache key) |
| `run_commands` | ✅ | — | Commands to run the scraper (multi-line supported) |
| `commit_paths` | — | `'*.csv'` | Files to git-add before commit |
| `needs_dmv_secrets` | — | `false` | Pass DMV_COOKIES + DMV_HEADERS as env vars |

Secrets passed to the template:
- `PRIVATE_REPO_TOKEN` (required) — PAT with repo scope
- `DMV_COOKIES` (optional) — only needed when `needs_dmv_secrets: true`
- `DMV_HEADERS` (optional) — only needed when `needs_dmv_secrets: true`

## Security considerations

- ✅ Private repo code is never copied into this public repo — it's checked out at runtime and discarded
- ✅ Secrets are stored in this repo's GitHub Secrets and are never exposed in logs
- ⚠️ **Workflow run logs are public** — anyone can see them. Make sure your scraper scripts don't `echo` or `print` sensitive values (API keys, cookies, etc.)
- ⚠️ The PAT has access to all your private repos. If it leaks, rotate immediately at https://github.com/settings/tokens

## Maintenance notes

- **VA cookies expire**: The `DMV_COOKIES` secret captures a browser session and will need to be refreshed periodically (often every few days). When the VA scraper starts failing, recapture cookies from DevTools → Network → Copy as cURL.
- **PAT expiration**: If you set the PAT to expire in 1 year, set a calendar reminder to rotate it before then — otherwise all workflows will fail simultaneously.
- **Adding new scrapers**: Just drop a new caller workflow in `.github/workflows/` following the pattern above. No need to modify the template.
