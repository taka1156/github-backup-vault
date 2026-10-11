# GitHub Backup Vault

[English Version](README.md)

GitHubリポジトリを月2回自動でバックアップし、Cloudflare R2 と Google ドライブに保存します。

`repos.txt` に書いたリポジトリ一覧を GitHub Actions が読み込み、全てミラークローンして1つのzipにまとめ、サイズ情報のCSVと一緒にR2とGoogleドライブへアップロードします。

## 仕組み

1. `repos.txt` を読み込みます（1行に1つ `owner/repo`）。
2. 各リポジトリを `git clone --mirror` で取得します（全ブランチ・タグを含みます）。
3. `github-backup-YYYYMMDD-HHMMSS.zip` にまとめます。
4. サイズ、ブランチ数、タグ数、コミット数、最終コミット日時を、ログ・ジョブサマリー・`github-backup-YYYYMMDD-HHMMSS-manifest.csv` に出力します。
5. [rclone](https://rclone.org/) で、zipとCSVをR2、Googleドライブにアップロードします。
6. 保持期間（`RETENTION_DAYS`、デフォルト30日）より古いバックアップを、送信先から削除します。
7. 一部のリポジトリのクローンに失敗しても、残りはバックアップされ、最後にワークフローが失敗扱いになります。

スケジュールは、毎月1日と15日の18:00 UTC（日本時間では2日と16日の午前2〜3時）です。手動実行もできます。

## 使い方

### 1. リポジトリを作る

新しくリポジトリを作り、次のファイルを置きます。

```
.github/workflows/backup-repos.yml
repos.txt
```

> **Private（非公開）をおすすめします。** 公開リポジトリでは、60日間リポジトリに動きがないとスケジュール実行が自動で無効化されます。また、リポジトリ名やサイズが公開ログに載ります。

### 2. バックアップするリポジトリを書く

`repos.txt` に1行1リポジトリで書きます。

```
# # から始まる行はコメントです
your-name/your-repo
your-org/another-repo
https://github.com/your-name/third-repo   # URL形式でも書けます
```

### 3. GitHubのトークンを作る

一覧にある全リポジトリを読めるPersonal Access Tokenを作ります。

- Fine-grained token：対象リポジトリを選び、**Contents: Read-only** を付けます（Metadataは自動で付きます）。
- Classic token：`repo` スコープ（プライベートリポジトリに必要です）。

作ったトークンは `GH_BACKUP_TOKEN` シークレットに保存します。

### 4. 送信先を用意する

片方だけでも、両方でも使えます。片方しか設定しない場合は、「手動で実行する」の手順で送信先を選んで実行してください。

#### Cloudflare R2

1. Cloudflareのダッシュボードでバケットを作ります（R2 → Create bucket）。
2. R2 → Manage R2 API Tokens で、そのバケットに **Object Read & Write** 権限のトークンを作ります。
3. アカウントID、Access Key ID、Secret Access Key を控えます。

バックアップはバケット内の `github-backup/` に保存されます。

#### Googleドライブ

1. Google Cloudでプロジェクトを作り、**Google Drive API** を有効にします。
2. **サービスアカウント**を作り、JSONキーをダウンロードします。
3. **共有ドライブ**内にフォルダを作り（またはマイドライブのフォルダでも可）、サービスアカウントのメールアドレスを編集者として共有します。
4. フォルダのURL（`.../folders/<FOLDER_ID>`）からフォルダIDを控えます。

> サービスアカウント自体にはストレージ容量がないため、共有ドライブを使うのが確実です。マイドライブだけを使う場合は、アップロードに失敗することがあります。その場合は、rcloneの設定をOAuth方式に変更してください。

### 5. シークレットを登録する

Settings → Secrets and variables → Actions → New repository secret で登録します。

| シークレット名 | 必要になる場面 | 内容 |
|---|---|---|
| `GH_BACKUP_TOKEN` | 常に必要 | 一覧の全リポジトリを読めるGitHubのPAT |
| `R2_ACCOUNT_ID` | R2 | CloudflareのアカウントID |
| `R2_ACCESS_KEY_ID` | R2 | R2のAPIトークンのアクセスキーID |
| `R2_SECRET_ACCESS_KEY` | R2 | R2のAPIトークンのシークレットアクセスキー |
| `R2_BUCKET` | R2 | バケット名 |
| `GDRIVE_SERVICE_ACCOUNT_JSON` | Googleドライブ | サービスアカウントのJSONキーの中身すべて |
| `GDRIVE_FOLDER_ID` | Googleドライブ | 送信先フォルダのID |
| `GDRIVE_SHARED_DRIVE_ID` | Googleドライブ（共有ドライブのみ） | 共有ドライブのID。通常のフォルダなら空のままにします |

### 6. 手動で実行する

Actions → **Backup repositories** → **Run workflow** を開き、送信先（`both` / `r2` / `gdrive`）を選びます。

スケジュール実行は常に `both` で動くため、定期実行を成功させるには両方のシークレットが必要です。片方だけ使う場合は、ワークフローから使わないほうのアップロード処理を削除してください。

### 7. 結果を確認する

実行ページの **Summary** を開くと、zipのサイズとリポジトリごとの表が確認できます。

## 復元方法

zipを展開すると、各リポジトリが `owner__repo.git` という名前のミラー形式で入っています。

```bash
unzip github-backup-YYYYMMDD-HHMMSS.zip -d backup
git clone backup/owner__repo.git restored-repo        # 作業用のコピーを取り出す
```

全ブランチ・タグを含む履歴を、空の新しいリモートにそのまま移す場合は次のとおりです。

```bash
cd backup/owner__repo.git
git push --mirror https://github.com/you/new-repo.git
```

## 設定

`.github/workflows/backup-repos.yml` の先頭にある `env:` を編集します。

| 変数 | デフォルト | 内容 |
|---|---|---|
| `REPO_LIST_FILE` | `repos.txt` | リポジトリ一覧のファイル |
| `RETENTION_DAYS` | `30` | この日数より古いバックアップを削除します。`0` で削除しません |

スケジュールを変えるには `cron` の行を編集します（GitHubはUTCで動きます）。

## バックアップされないもの

含まれるのはgitのデータ（コミット、ブランチ、タグ）だけです。Issue、プルリクエスト、Wiki、リリース、Actionsの設定は含まれません。

## 注意

- zipが送信先の容量に収まるか確認してください。ランナーのディスクにも限りがあります（標準ランナーで空き約14GB）。
- R2の古いバックアップの削除は、`github-backup/` 内の `RETENTION_DAYS` より古いファイルをすべて消します。関係のないファイルを置かないでください。