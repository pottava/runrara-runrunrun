# Cloud Run と Buildpacks によるサーバーレス開発

<walkthrough-watcher-constant key="region" value="asia-northeast1"></walkthrough-watcher-constant>
<walkthrough-watcher-constant key="zone" value="asia-northeast1-c"></walkthrough-watcher-constant>

## 始めましょう

[Cloud Run(https://cloud.google.com/run?hl=ja) と [Buildpacks](https://github.com/GoogleCloudPlatform/buildpacks) によるサーバーレス開発を体験する手順です。

**所要時間**: 約 5 分

**前提条件**:

- 計算クラスタのためのプロジェクトが作成してある
- プロジェクトのオーナー権限をもつアカウントでログインしている

**[開始]** ボタンをクリックして次のステップに進みます。

## プロジェクトの設定

ハンズオンに利用するプロジェクトを選択してください。

<walkthrough-project-billing-setup permissions="run.googleapis.com"></walkthrough-project-billing-setup>

## デフォルト値の設定

コマンド実行時のデフォルト値を設定します。

```sh
gcloud config set project "{{project-id}}"
gcloud config set run/platform managed
gcloud config set run/region {{region}}
```

続いて、Cloud Run の API を有効化します。

```sh
gcloud services enable run.googleapis.com cloudbuild.googleapis.com
```

## アプリケーションのデプロイ

アプリケーションをビルドし、GCR に push します。

```sh
cd buildpacks
gcloud builds submit --pack image="gcr.io/{{project-id}}/my-app:v0.1"
```

デプロイします。

```sh
gcloud run deploy my-app --image "gcr.io/{{project-id}}/my-app:v0.1" \
    --allow-unauthenticated
```

起動したら接続してみましょう。

```sh
gcloud run services describe my-app --format "value(status.url)"
```

## アプリケーションの変更と no traffic でのデプロイ

Editor を起動し、アプリケーションを書き換えてみましょう。

<walkthrough-editor-open-file filePath="~/cloudshell_open/runrara-runrunrun/buildpacks/index.js">Editor を起動</walkthrough-editor-open-file>

アプリケーションをビルドし、GCR に push します。

```sh
gcloud builds submit --pack image="gcr.io/{{project-id}}/my-app:v0.2"
```

トラフィックを流さない設定でデプロイします。

```sh
gcloud run deploy my-app --image "gcr.io/{{project-id}}/my-app:v0.2" --no-traffic 
```

## カナリーデプロイとロールバック

サービスの状況を確認してみましょう。

```sh
gcloud run services describe my-app
```

最新リビジョンにトラフィックの 20 % を流してみます。

```sh
gcloud run services update-traffic my-app --to-revisions=LATEST=20
gcloud run services describe my-app --format "json(spec.traffic)"
```

元のリビジョンに戻しましょう。

```sh
gcloud run services update-traffic my-app --to-revisions=LATEST=0
```
