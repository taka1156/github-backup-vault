## GitHub Backup Vault

[日本語](README.ja.md)

Automated twice-monthly backup of my GitHub repos to Cloudflare R2 and Google Drive.

A GitHub Actions workflow reads a list of repositories from `repos.txt`, mirror-clones each one, packs everything into a single zip, and uploads it (plus a CSV report with sizes) to Cloudflare R2 and/or Google Drive.

## How it works

1. Reads `repos.txt` (one `owner/repo` per line).
2. Runs `git clone --mirror` for each repository (all branches and tags are included).
3. Zips everything into `github-backup-YYYYMMDD-HHMMSS.zip`.
4. Writes a report (size, branches, tags, commits, last commit date per repo) to the log, the job summary, and `github-backup-YYYYMMDD-HHMMSS-manifest.csv`.
5. Uploads the zip and the CSV to R2 and/or Google Drive with [rclone](https://rclone.org/).
6. Deletes backups older than `RETENTION_DAYS` (default: 30) at the destinations.
7. If any repository failed to clone, the others are still backed up and the run is marked as failed at the end.

Schedule: the 1st and 15th of every month at 18:00 UTC (02:00/03:00 JST on the 2nd and 16th). You can also run it manually.

## Usage

### 1. Create the repository

Create a new repository and put these files in it:

```
.github/workflows/backup-repos.yml
repos.txt
```

> Tip: make it **Private**. In public repositories, scheduled workflows are disabled automatically after 60 days without repository activity, and repo names and sizes would appear in public logs.

### 2. List the repositories to back up

Edit `repos.txt`, one repository per line:

```
# comments start with #
your-name/your-repo
your-org/another-repo
https://github.com/your-name/third-repo   # URLs also work
```

### 3. Create a GitHub token

Create a Personal Access Token that can read every repository in the list.

- Fine-grained token: select the repositories, and set **Contents: Read-only** (Metadata is added automatically).
- Classic token: the `repo` scope (needed for private repositories).

Save it as the `GH_BACKUP_TOKEN` secret.

### 4. Set up the destinations

You can use either one or both. If a destination's secrets are not set, run the workflow manually and pick the other destination (see "Run manually").

#### Cloudflare R2

1. Create a bucket in the Cloudflare dashboard (R2 → Create bucket).
2. R2 → Manage R2 API Tokens → create a token with **Object Read & Write** for that bucket.
3. Note the Account ID, Access Key ID, and Secret Access Key.

Backups are stored under `github-backup/` in the bucket.

#### Google Drive

1. In Google Cloud, create a project and enable the **Google Drive API**.
2. Create a **service account** and download its JSON key.
3. Create a folder in a **shared drive** (or a folder in My Drive) and share it with the service account's email as an editor.
4. Copy the folder ID from the folder URL (`.../folders/<FOLDER_ID>`).

> Service accounts have no storage quota of their own, so a shared drive is the most reliable choice. If you only use My Drive, a service account may fail to upload; in that case switch the rclone config to OAuth.

### 5. Add the secrets

Settings → Secrets and variables → Actions → New repository secret.

| Secret | Required for | Description |
|---|---|---|
| `GH_BACKUP_TOKEN` | always | GitHub PAT with read access to all listed repos |
| `R2_ACCOUNT_ID` | R2 | Cloudflare account ID |
| `R2_ACCESS_KEY_ID` | R2 | R2 API token access key ID |
| `R2_SECRET_ACCESS_KEY` | R2 | R2 API token secret access key |
| `R2_BUCKET` | R2 | Bucket name |
| `GDRIVE_SERVICE_ACCOUNT_JSON` | Google Drive | Full contents of the service account JSON key |
| `GDRIVE_FOLDER_ID` | Google Drive | Destination folder ID |
| `GDRIVE_SHARED_DRIVE_ID` | Google Drive (shared drive only) | Shared drive ID. Leave empty for a normal folder |

### 6. Run manually

Actions → **Backup repositories** → **Run workflow**, then choose the destination: `both`, `r2`, or `gdrive`.

Scheduled runs always use `both`, so both sets of secrets must be configured for scheduled runs to succeed. If you only want one destination, delete the other upload step from the workflow.

### 7. Check the result

Open the run and look at the **Summary** tab. It shows the zip size and a per-repository table.

## Restore

Unzip the archive. Each repository is a bare mirror named `owner__repo.git`.

```bash
unzip github-backup-YYYYMMDD-HHMMSS.zip -d backup
git clone backup/owner__repo.git restored-repo        # get a working copy
```

To push the full history (all branches and tags) to a new empty remote:

```bash
cd backup/owner__repo.git
git push --mirror https://github.com/you/new-repo.git
```

## Settings

Edit the `env:` block at the top of `.github/workflows/backup-repos.yml`.

| Variable | Default | Description |
|---|---|---|
| `REPO_LIST_FILE` | `repos.txt` | File that lists the repositories |
| `RETENTION_DAYS` | `30` | Delete backups older than this many days. `0` disables deletion |

To change the schedule, edit the `cron` line (GitHub uses UTC).

## What is not backed up

Only git data (commits, branches, tags) is included. Issues, pull requests, wikis, releases, and Actions settings are not.

## Notes

- Make sure the zip is small enough for your destinations. The runner has a limited disk (about 14 GB free on standard runners).
- Deleting old backups at R2 removes everything older than `RETENTION_DAYS` under `github-backup/`, so do not store unrelated files there.
