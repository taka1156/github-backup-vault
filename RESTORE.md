# Restore Guide

[日本語はこちら](RESTORE.ja.md)

Each backup zip (`github-backup-YYYYMMDD-HHMMSS.zip`) contains one bare mirror per repository, stored as `owner__repo.git`. A matching `github-backup-YYYYMMDD-HHMMSS-manifest.csv` lists size, branch/tag/commit counts and the last commit date for each repository.

### 1. Get the zip

**Cloudflare R2** (via rclone, or download from the Cloudflare dashboard):

```bash
rclone copyto R2:<bucket>/github-backup/github-backup-20261011-180000.zip ./backup.zip
```

**Google Drive**: download from the browser.

Extract everything, or only the repository you need:

```bash
unzip backup.zip -d restore
# or just one repository
unzip backup.zip 'owner__repo.git/*' -d restore
```

### 2. Restore

**A. Get a normal working copy (inspect files, resume work)**

```bash
git clone restore/owner__repo.git owner-repo
```

**B. Restore to GitHub**

1. Create an empty repository on GitHub (do not add a README, license or .gitignore).
2. Push branches and tags from the mirror:

```bash
cd restore/owner__repo.git
git push --prune https://github.com/owner/repo.git \
  '+refs/heads/*:refs/heads/*' '+refs/tags/*:refs/tags/*'
```

Do not use `git push --mirror`: the mirror contains `refs/pull/*`, which GitHub rejects as hidden refs and the push reports errors. Pushing only branches and tags avoids this.

### 3. What is NOT restored

- Issues, pull request conversations and comments, wikis, releases, labels, milestones, and other GitHub-side metadata are not part of this backup.
- Git LFS objects are not included in a `git clone --mirror`. If a repository uses LFS, the LFS files need a separate `git lfs fetch --all` step in the backup workflow.

