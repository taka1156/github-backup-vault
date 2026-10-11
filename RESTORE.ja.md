# Restore Guide

[English Version](RESTORE.md) 

バックアップのzip（`github-backup-YYYYMMDD-HHMMSS.zip`）には、各リポジトリのベアミラーが `owner__repo.git` という名前で入っています。同じ日時の `github-backup-YYYYMMDD-HHMMSS-manifest.csv` には、各リポジトリのサイズ、ブランチ数・タグ数・コミット数、最終コミット日時が記録されています。

### 1. zipを取得する

**Cloudflare R2**（rcloneか、Cloudflareのダッシュボードからダウンロード）:

```bash
rclone copyto R2:<バケット名>/github-backup/github-backup-20261011-180000.zip ./backup.zip
```

**Googleドライブ**: ブラウザからダウンロードします。

すべて展開するか、必要なリポジトリだけ展開します。

```bash
unzip backup.zip -d restore
# 1リポジトリだけ展開する場合
unzip backup.zip 'owner__repo.git/*' -d restore
```

### 2. 復元する

**A. 通常の作業コピーとして取り出す（中身の確認、作業の再開）**

```bash
git clone restore/owner__repo.git owner-repo
```

**B. GitHubに復元する**

1. GitHubで空のリポジトリを作成します（README、ライセンス、.gitignoreは付けません）。
2. ミラーからブランチとタグをプッシュします。

```bash
cd restore/owner__repo.git
git push --prune https://github.com/owner/repo.git \
  '+refs/heads/*:refs/heads/*' '+refs/tags/*:refs/tags/*'
```

`git push --mirror` は使わないでください。ミラーに含まれる `refs/pull/*` はGitHubが非公開の参照として拒否するため、エラーになります。ブランチとタグだけをプッシュすれば回避できます。

### 3. 復元できないもの

- Issue、PRの本文やコメント、Wiki、Release、ラベル、マイルストーンなど、GitHub上のメタデータはこのバックアップに含まれません。
- Git LFSのオブジェクトは `git clone --mirror` には含まれません。LFSを使っているリポジトリは、バックアップのワークフローに `git lfs fetch --all` を追加する必要があります。