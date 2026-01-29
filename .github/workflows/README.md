# GitHub Actions Workflows 説明書

このディレクトリ（`.github/workflows`）に含まれるワークフローファイルの機能、使い方、注意点について解説します。

## 目次

1. [Unity Build (build-unity.yml)](#unity-build-build-unityyml)
2. [Deploy to GitHub Pages (deploy-github-pages.yml)](#deploy-to-github-pages-deploy-github-pagesyml)
3. [Universal Discord Notifier (discord-notifier.yml)](#universal-discord-notifier-discord-notifieryml)
4. [GitHub Event Handler (event-handler.yml)](#github-event-handler-event-handleryml)
5. [Release To GitHub Release (release-github.yml)](#release-to-github-release-release-githubyml)
6. [Deploy WebGL Build To Netlify (deploy-netlify.yml)](#deploy-webgl-build-to-netlify-deploy-netlifyyml)
7. [Create GitHub Release Note (create-release-note.yml)](#create-github-release-note-create-release-noteyml)
8. [Push Build To Other Repository (push-build-to-other-repo.yml)](#push-build-to-other-repository-push-build-to-other-repoyml)
9. [全体的な注意点](#全体的な注意点)

---

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

---

## Deploy to GitHub Pages (deploy-github-pages.yml)

### 機能
UnityのWebGLビルドを作成し、GitHub Pagesにデプロイします。
デプロイ完了時にDiscordへ通知を送信します。

### 使い方
- **他ワークフローからの呼び出し (workflow_call):**
  - 引数はなく、デフォルトで `For_GitHub_Pages` モードでビルド・デプロイが行われます。

### 注意点
- リポジトリの `Settings > Pages` で、Sourceを `GitHub Actions` に設定している必要があります。
- ビルド部分で `build-unity.yml` を呼び出しています。

---

## Universal Discord Notifier (discord-notifier.yml)

### 機能
Discordに通知を送信するための汎用ワークフローです。

| モード | トリガー | 通知内容の特徴 |
| :--- | :--- | :--- |
| **自動実行** | 全ワークフロー完了時 | 内容は自動生成。**PRリンク**があれば自動追加されます。 |
| **手動呼び出し** | `workflow_call` | タイトル・本文を自由に指定可能。**任意のURL** (`target_url`) を追加できます。 |

### 使い方
- **自動実行:** 設定不要。すべてのワークフロー完了時に動作します。
- **他ワークフローからの呼び出し (workflow_call):**
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

---

## GitHub Event Handler (event-handler.yml)

### 機能
Pull Requestがクローズ（マージ）された際のイベントハンドリングを行います。
PRに付与されたラベル（`release/Windows` や `release/WebGL`）に応じて、リリース処理やデプロイ処理に振り分けます。

### 使い方
- **自動実行:** `main` ブランチへのPRがクローズされた時にトリガーされます。
- ブランチ名が `release/1.0.0` のような形式の場合、バージョンタグ `1.0.0` を抽出して後続ジョブに渡します。

### 注意点
- **重要:** 現在、コード内で `uses: ./.github/workflows/release-netlify.yml` を参照していますが、ファイルの実体は `deploy-netlify.yml` である可能性があります。ファイル名の不整合によりエラーになる恐れがあります。

---

## Release To GitHub Release (release-github.yml)

### 機能
Unityビルドを実行し、その成果物をZIP化してGitHub Releasesにアップロード（リリース作成）します。

### 使い方
- **手動実行 (workflow_dispatch):**
  - ビルドターゲットとバージョンタグを指定して実行。
- **他ワークフローからの呼び出し:**
  - `event-handler.yml` などから呼び出されます。

---

## Deploy WebGL Build To Netlify (deploy-netlify.yml)

### 機能
Unity WebGLビルドを行い、別のリポジトリ（Netlify連携用リポジトリ想定）に成果物をプッシュしてデプロイします。

### 使い方
- **手動実行 (workflow_dispatch):**
  - バージョンタグを指定して実行。
- **他ワークフローからの呼び出し:**
  - バージョンタグを渡して実行。

### 注意点
- 実際には `push-build-to-other-repo.yml` を呼び出してデプロイ処理を行います。

---

## Create GitHub Release Note (create-release-note.yml)

### 機能
ビルド成果物を受け取り、ZIP圧縮してGitHub Releasesを作成するヘルパーワークフローです。

### 使い方
- `release-github.yml` などの上位ワークフローから呼び出されて使用されます。単体で使うことはあまりありません。

---

## Push Build To Other Repository (push-build-to-other-repo.yml)

### 機能
ビルド成果物をダウンロードし、指定された別のGitリポジトリにコミット＆プッシュします。

### 使い方
- `deploy-netlify.yml` などから呼び出されます。
- **Secrets:** デプロイ先リポジトリへのアクセス権限を持つ `DEPLOY_REPO_SECRET_KEY` (SSH鍵) が必要です。

---

## 全体的な注意点

1. **ファイル名の不整合:**
   - `event-handler.yml` が参照している `release-netlify.yml` は存在せず、おそらく `deploy-netlify.yml` の間違いです。修正が必要です。

2. **Secretsの管理:**
   - 多くのワークフローで `secrets: inherit` を使用しています。呼び出し元のリポジトリ設定で `UNITY_EMAIL`, `UNITY_PASSWORD`, `UNITY_LICENSE`, `DISCORD_WEBHOOK`, `DEPLOY_REPO_SECRET_KEY` などが正しく設定されていることを確認してください。
