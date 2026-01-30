# GitHub Actions Workflows 説明書

このディレクトリ（`.github/workflows`）に含まれるワークフローファイルの機能、使い方、注意点について解説します。

## 目次

1. [必要な設定](#必要な設定)
1. [Unity Build (build-unity.yml)](#unity-build-build-unityyml)
1. [Deploy to GitHub Pages (deploy-github-pages.yml)](#deploy-to-github-pages-deploy-github-pagesyml)
1. [Universal Discord Notifier (discord-notifier.yml)](#universal-discord-notifier-discord-notifieryml)
1. [Release To GitHub Release (release-github.yml)](#release-to-github-release-release-githubyml)
1. [Push Build To Other Repository (push-build-to-other-repo.yml)](#push-build-to-other-repository-push-build-to-other-repoyml)

---

## 必要な設定
### Unityのライセンス関連
* `UNITY_EMAIL`: Unityアカウントのメールアドレス
* `UNITY_PASSWORD`: Unityアカウントのパスワード
* `UNITY_LICENSE`:
  1. UnityHubを開く
  1. LICENSEタブを開く
  1. Add licenseを押す
  1. Get a free personal licenseを押す
  1. Agree and get personal edition licenseを押す
  1. `C:\ProgramData\Unity\Unity_lic.ulf`の中身をコピー


### DiscordのウェブフックURL
* `DISCORD_WEBHOOK`:
  1. Discordのサーバー設定
  1. 連携サービス
  1. ウェブフック
  1. 新しいウェブフック
  1. 名前とアイコンはワークフロー側で勝手に変えるから適当でOK
  1. ウェブフックURLをコピー

### 設定方法
1. GitHubリポジトリのSettings
1. Secrets and variables -> Actions
1. Secretsタブ -> Repository secrets
1. New repository secret

## Unity Build (build-unity.yml)

### 機能
Unityプロジェクトのビルドを行う中核となるワークフローです。Windows向けビルドとWebGL向けビルドに対応しています。
GitHub Pages向けにWebGLビルドの設定を自動調整する機能も含まれています。

### 使い方
- **手動実行 (workflow_dispatch):**
  - Actionsタブから「Unity Build (Windows/WebGL)」を選択。
  - `build_target_name`: `Windows`, `WebGL`, `For_GitHub_Pages` から選択して実行。
- **他ワークフローからの呼び出し (workflow_call):**
  - `uses: ./.github/workflows/build-unity.yml` で呼び出し可能。
  - `secrets: inherit` でSecretsを引き継ぐ必要があります。

### 注意点
- **Secrets:** `UNITY_EMAIL`, `UNITY_PASSWORD`, `UNITY_LICENSE` が必須です。
- **キャッシュ:** LibraryとPackagesのキャッシュを利用してビルド時間を短縮しています。

## Deploy to GitHub Pages (deploy-github-pages.yml)

### 機能
UnityのWebGLビルドを作成し、GitHub Pagesにデプロイします。
デプロイ完了時にDiscordへ通知を送信します。

### 使い方
- **自動実行 (push):**
  - `main` ブランチへプッシュされた時に実行されます。
- **他ワークフローからの呼び出し (workflow_call):**
  - 引数はなく、デフォルトで `For_GitHub_Pages` モードでビルド・デプロイが行われます。

### 注意点
- リポジトリの `Settings > Pages` で、Sourceを `GitHub Actions` に設定している必要があります。
- ビルド部分で `build-unity.yml` を呼び出しています。

## Universal Discord Notifier (discord-notifier.yml)

### 機能
Discordに通知を送信するための汎用ワークフローです。
他ワークフローから呼び出された時のみ実行されます。PRに関連する場合、通知にPRへのリンクを自動で付与します。

### 使い方
- **他ワークフローからの呼び出し (workflow_call):**
  通知を行いたいワークフローの `jobs` に以下を追加します。
  ```yaml
  - uses: ./.github/workflows/discord-notifier.yml
    with:
      title: "タイトル"
      message: "メッセージ本文"
      status: "success" # success / failure / warn
      target_url: "https://..." # 遷移先URL（任意）
    secrets: inherit
  ```

### 注意点
- **Secrets:** `DISCORD_WEBHOOK` が必要です。

## Release To GitHub Release (release-github.yml)

### 機能
Unityビルドを実行し、その成果物をZIP化してGitHub Releasesにアップロード（リリース作成）します。

### 使い方
- **手動実行 (workflow_dispatch):**
  - ビルドターゲットとバージョンタグを指定して実行。
- **他ワークフローからの呼び出し:**
  - `event-handler.yml` などから呼び出されます。


## Create GitHub Release Note (create-release-note.yml)

### 機能
ビルド成果物を受け取り、ZIP圧縮してGitHub Releasesを作成するヘルパーワークフローです。

### 使い方
- `release-github.yml` などの上位ワークフローから呼び出されて使用されます。単体で使うことはあまりありません。

## Push Build To Other Repository (push-build-to-other-repo.yml)

### 機能
ビルド成果物をダウンロードし、指定された別のGitリポジトリにコミット＆プッシュします。

### 使い方
- `deploy-netlify.yml` などから呼び出されます。
- **Secrets:** デプロイ先リポジトリへのアクセス権限を持つ `DEPLOY_REPO_SECRET_KEY` (SSH鍵) が必要です。
