# LocaleSync — Keep iOS and Android Strings Aligned

LocaleSync keeps localized strings in sync across **iOS and Android** from a single source of truth. It's an n8n workflow that reads every string from one shared Google Sheet and, on a schedule, generates the native files for each platform: `strings.xml` for Android and `Localizable.strings` for iOS. It pushes them to both GitLab repositories in the same run.

Both apps always ship the same copy. Translators edit one sheet, and neither platform drifts from the other.

## What it does

- Reads localization strings (en + zh-TW) from a shared Google Sheet
- Generates `strings.xml` for Android and `.strings` files for iOS
- Pushes to both GitLab repos in parallel via a daily branch (`string-update-YYYYMMDD`)
- Detects changes before committing — skips if content is identical (idempotent)
- Handles same-day reruns: reuses the existing branch, appends a new commit only if the sheet changed

## Architecture

| Workflow | Role |
|----------|------|
| `LocaleSync.json` | Sub-workflow — does the actual sync. Stays **inactive**; called by the parent. |
| `LocaleSync_Parent.json` | Scheduler and project config holder. **Activate this one.** |

One Set node per project in the parent holds 7 required config fields (sheet ID, repo paths, etc.).

## Prerequisites

- Docker with Docker Compose
- A Google Service Account with Google Sheets API and Google Drive API enabled
- A GitLab personal access token with `api` scope
- n8n running locally or on a server

## Setup

### 1. Environment

Generate a runner auth token and store both secrets in a `.env` file (never commit this):

```bash
# Generate N8N_RUNNERS_AUTH_TOKEN
openssl rand -hex 32
```

```env
# .env
N8N_RUNNERS_AUTH_TOKEN=your-generated-token-here
GITLAB_PRIVATE_TOKEN=your-gitlab-token-here
```

### 2. Start containers

```bash
docker compose up -d
```

Open `http://localhost:5678` and complete the owner account setup on first run.

### 3. Set up credentials in n8n

**Google Sheets (Service Account — recommended for automation)**

1. Create a Google Service Account key (JSON), mount it into the container as a volume
2. In the n8n UI, create a Google Sheets credential referencing the file path
3. Name it exactly: `Google Sheets account`

Reference: https://docs.n8n.io/integrations/builtin/credentials/google/#oauth2-and-service-account

**GitLab (HTTP Header Auth)**

1. In the n8n UI, create a Header Auth credential:
   - Name: `PRIVATE-TOKEN`
   - Value: `{{ $env.GITLAB_PRIVATE_TOKEN }}`
2. Name the credential: `Header Auth account`

The token is read from the container environment at runtime — no hardcoding needed.

Reference: https://docs.n8n.io/integrations/builtin/credentials/httprequest/

### 4. Import workflows (order matters)

1. Import `LocaleSync.json` — the sub-workflow. Leave it **inactive**.
   - Copy its workflow ID from the URL bar: `/workflow/<ID>`
2. Import `LocaleSync_Parent.json` — the scheduler and config holder.
   - Open each **Run for Project** Execute Workflow node and paste the ID from step 1.

After import, re-link credentials on any nodes showing a warning:
- `Google Sheets account` → Google Sheets nodes
- `Header Auth account` → all HTTP Request nodes

### 5. Configure projects in the parent workflow

Each project has one Set node (e.g. "Project A Config"). Fill in the 7 fields for each project — field descriptions are in the sticky note inside `LocaleSync`.

**Finding `sheet_id` and `sheet_gid`:** open the target Google Sheet and read the URL:

```
https://docs.google.com/spreadsheets/d/<sheet_id>/edit#gid=<sheet_gid>
```

Each tab has its own `gid`. Switch to the target tab first, then read it from the URL.

To add a project: duplicate an existing Config Set node, update its values, and wire it:  
`Schedule Trigger → new Config node → Run LocaleSync`

### 6. Prepare platform repos

**Android:** extract non-translatable strings to a separate strings file so the workflow only touches the translatable ones.

**iOS:** annotate sections in your `Localizable.strings` file with these headers:

```
// ========================================
// Skip Keys (auto-gen sync excluded)
// ========================================

// ========================================
// Localized Keys
// ========================================
```

Keys under **Skip Keys** are excluded from sync unless their value changed.

### 7. Set the schedule

Open the **Schedule Trigger** node in `LocaleSync_Parent` and set the desired run time.

### 8. Activate

Activate `LocaleSync_Parent`. `LocaleSync` stays inactive — it is called by the parent.

## Output Files

**Android**

| File | Language |
|------|----------|
| `app/src/main/res/values/strings.xml` | English |
| `app/src/main/res/values-zh-rTW/strings.xml` | Traditional Chinese |

**iOS**

| File | Language |
|------|----------|
| `en.lproj/Localizable.strings` | English |
| `zh-Hant.lproj/Localizable.strings` | Traditional Chinese |

## Google Sheet Column Format

| Column | Android | iOS | Description |
|--------|:-------:|:---:|-------------|
| `android name` | Yes | No | Android string resource key |
| `iOS name` | No | Yes | iOS localization key |
| `value(en)` | Yes | Yes | English value |
| `value(zh-TW)` | Yes | Yes | Traditional Chinese value |

## Change Detection

On each run, both flows compare generated content against what is already on the branch:

- **First run of the day** (branch doesn't exist): branch created, commit pushed immediately
- **Same-day rerun, no sheet changes**: commit skipped
- **Same-day rerun, sheet updated**: new commit appended to the existing branch

## Validation

On each run, LocaleSync validates all 7 config fields before touching any external service. A missing field halts the run with a descriptive error. The GitLab token is verified (`GET /user`) in parallel with the sheet read — a 401 means `GITLAB_PRIVATE_TOKEN` in `.env` is wrong or missing.
