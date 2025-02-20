---
title: "Cloud RunへのCI/CD実装：GitHub Actionsを用いた自動デプロイ"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["googlecloud", "githubactions", "cicd"]
published: true
publication_name: "team_delta"
---
## はじめに
従来はローカル環境からCloud Runへのデプロイを行っていました（このプロセス自体は比較的容易）が、やっぱりコミットプッシュしたら自動でデプロイして欲しいなーと思い、備忘録としてその方法を記述して行きます。

## 結論
```bash
# Artifact Registryでのリポジトリの作成
$ gcloud artifacts repositories create $REPOSITORY_NAME --repository-format=docker --location=asia-northeast1 

# gcloudでDockerが使えるように認証設定
$ gcloud auth configure-docker asia-northeast1-docker.pkg.dev

# GitHub Actions用のサービスアカウントの作成
$ gcloud iam service-accounts create $SA_NAME --display-name="github-actions"

# 作成したサービスアカウントに必要な権限を付与
$ gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com" --role="roles/run.admin"
$ gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com" --role="roles/iam.serviceAccountUser"
$ gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com" --role="roles/artifactregistry.writer"

# IAM Credentials APIの有効化
$ gcloud services enable iamcredentials.googleapis.com --project=$PROJECT_ID

# Workload Identity プールの作成
$ gcloud iam workload-identity-pools create $POOL_NAME --location="global" --project=$PROJECT_ID

# Workload Identity プロバイダ（OIDCプロバイダ）の作成
$ gcloud iam workload-identity-pools providers create-oidc $PROVIDER_NAME \
    --project="$PROJECT_ID" \
    --location="global" \
    --workload-identity-pool="$POOL_NAME" \
    --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
    --issuer-uri="https://token.actions.githubusercontent.com" \
    --attribute-condition="assertion.repository_owner=='$YOUR_ORGANIZATION/$YOUR_REPOSITORY'"

# サービスアカウントとWorkload Identity プールの紐付け
$ gcloud iam service-accounts add-iam-policy-binding $SA_NAME@$PROJECT_ID.iam.gserviceaccount.com \
  --project="$PROJECT_ID" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/$PROJECT_NUM/locations/global/workloadIdentityPools/$PROVIDER_NAME/attribute.repository/$YOUR_ORGANIZATION/$YOUR_REPOSITORY"
```

```yaml:.github/workflows/cloudrun.yml
on:
  push:
    branches: [main]  # デプロイ対象のブランチを指定

jobs: 
  prod: 
    runs-on: ubuntu-latest
    permissions: 
      id-token: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Google Auth
        id: auth
        uses: 'google-github-actions/auth@v2'
        with:
          project_id: '$PROJECT_ID'
          workload_identity_provider: 'projects/$PROJECT_NUM/locations/global/workloadIdentityPools/$PROVIDER_NAME/providers/github'
          token_format: 'access_token'
          service_account: '$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com'
      - name: Docker Auth
        id: docker-auth
        uses: 'docker/login-action@v3'
        with:
          username: 'oauth2accesstoken'
          password: '${{ steps.auth.outputs.access_token }}'
          registry: 'asia-northeast1-docker.pkg.dev'
      - name: Build, tag and push container
        id: build-image
        uses: docker/build-push-action@v6
        with:
          context: './src/' # アプリケーションのソースコードが配置されているディレクトリ
          push: true 
          tags: |
             asia-northeast1-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/$IMAGE_NAME:${{ github.sha }}
             asia-northeast1-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/$IMAGE_NAME:latest
      - id: 'deploy'
        uses: 'google-github-actions/deploy-cloudrun@v2'
        with:
          service: '$SERVICE_NAME'
          region: 'asia-northeast1'
          image: 'asia-northeast1-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/$IMAGE_NAME:latest'
```

## 準備１：プロジェクトIDとプロジェクト番号の確認
本記事ではプロジェクトIDとプロジェクト番号が必要になります。
これらはGoogle Cloud Consoleにアクセスし、新規プロジェクトを作成後に表示されます。

![](/images/google_cloud_1/console.png)
*Google Cloud Console*

コマンドからでしたらプロジェクトIDは以下のコマンドで確認できます（gcloud CLIは事前にDL必要！）：
```bash
$ gcloud projects list
```
また、本記事ではプロジェクトIDとプロジェクト番号に関しては```$PROJECT_ID```と```$PROJECT_NUM```とします。

## 準備2：Artifact Registryでのリポジトリの作成
Artifact Registryは、Google Cloudが提供するフルマネージドのコンテナイメージレジストリサービスです。Dockerイメージ、Node.jsパッケージ、Javaライブラリ、Pythonパッケージなど、さまざまな種類のソフトウェアアーティファクトを保存、管理、セキュアに配布することができます。
これは従来のContainer Registryの後継サービスとして位置づけられています。
ローカル環境からデプロイするにあたってもArtifact Registryでリポジトリを作成する必要があります。
今回はdockerイメージを対象にしているので、作成にあたっては以下のコマンドを実行してください：
```bash
$ gcloud artifacts repositories create $REPOSITORY_NAME --repository-format=docker --location=asia-northeast1 
```
```$REPOSITORY_NAME```はArtifact Registryのリポジトリ名ですので、好きな名前を設定してください。

作成したリポジトリの一覧を確認するには以下のコマンドで確認してください：
```bash
$ gcloud artifacts repositories list
```

最後に以下のコマンドでgcloudでDockerが使えるよう認証することを忘れずに：
```bash
$ gcloud auth configure-docker asia-northeast1-docker.pkg.dev
```
```asia-northeast1-docker.pkg.dev```は、Artifact Registryにおける、東京リージョンのエンドポイントです。

## ローカル環境からのデプロイ
そもそもローカル環境からどのようにデプロイするかの方法を記します。
結論簡単で、以下の数少ないコマンドを実行すれば完了します。
このアーキテクチャ図が参考にあるとわかりやすいです：[Cloud Run サービスのデプロイのよくあるパターン 3 選](https://zenn.dev/google_cloud_jp/articles/cloudrun-deploy-pattern#github-%E3%83%95%E3%83%AB%E6%B4%BB%E7%94%A8%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3)

1. Dockerイメージのビルド
```bash
$ docker buildx build --platform linux/amd64 ./ -t asia-northeast1-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/$IMAGE_NAME:latest
```
```buildx```はマルチプラットフォーム対応のDockerイメージをビルドするための拡張機能です。
```--platform linux/amd64```により、Cloud Run上で動作する適したアーキテクチャでイメージをビルドすることを指定しています。特にM1/M2チップのMacのような異なるアーキテクチャの開発環境からデプロイする際はこのオプションをつけてください。
```$IMAGE_NAME```は好きなイメージ名にしてください。

2. DockerイメージをArtifact Registryにプッシュ
```bash
$ docker push asia-northeast1-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/$IMAGE_NAME:latest
```

3. Cloud Runへのデプロイ
```bash
$ gcloud run deploy $SERVICE_NAME \
    --image asia-northeast1-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/$IMAGE_NAME:latest \
    --region asia-northeast1 \
    --platform managed \
    --port 5000
```
```--platform managed```により、フルマネージドなサーバーレス環境での実行を指定し、```--port 5000```でコンテナがリッスンするポート番号を指定しています。
これらの設定は必要に応じて変更したり削除したりしてください。
```$SERVICE_NAME```は好きなサービスの名前を設定してください。Cloud Run上でアプリケーションを識別するための一意の名前になります。

## GitHub Actions用の認証設定
ローカル環境からのデプロイは簡単でしたが、毎回あれらのコマンドを叩いてデプロイするのも面倒だと思うので自動化する設定を行いたいと思います。
1. GitHub Actions用のサービスアカウントの作成
```bash
$ gcloud iam service-accounts create $SA_NAME --display-name="github-actions"
```
```$SA_NAME```は好きなサービスアカウントの名前にしてください。

サービスアカウントが作成されたことを確認するには以下のコマンドを実行してください：
```bash
$ gcloud iam service-accounts list
```

2. 作成したサービスアカウントに必要な権限を付与
```bash
$ gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com" --role="roles/run.admin"
$ gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com" --role="roles/iam.serviceAccountUser"
$ gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com" --role="roles/artifactregistry.writer"
```

3. Workload Identity Federationの設定
まずIAM Credentials APIの有効化を行います：
```bash
$ gcloud services enable iamcredentials.googleapis.com --project=$PROJECT_ID
```

次にWorkload Identity プールの作成を行います：
```bash
$ gcloud iam workload-identity-pools create $POOL_NAME --location="global" --project=$PROJECT_ID
```
```$POOL_NAME```は好きなプール名を設定してください。

Workload Identity プロバイダ（OIDCプロバイダ）の作成は以下のコマンドで行います：
```bash
$ gcloud iam workload-identity-pools providers create-oidc $PROVIDER_NAME \
    --project="$PROJECT_ID" \
    --location="global" \
    --workload-identity-pool="$POOL_NAME" \
    --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
    --issuer-uri="https://token.actions.githubusercontent.com" \
    --attribute-condition="assertion.repository_owner=='$YOUR_ORGANIZATION/$YOUR_REPOSITORY'"
```
```$PROVIDER_NAME```は好きなプロバイダ名にしてください。
```$YOUR_ORGANIZATION/$YOUR_REPOSITORY```は実際のGitHubリポジトリ名に置き換えてください。

最後にサービスアカウントとWorkload Identity プールの紐付けを行います：
```bash
$ gcloud iam service-accounts add-iam-policy-binding $SA_NAME@$PROJECT_ID.iam.gserviceaccount.com \
  --project="$PROJECT_ID" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/$PROJECT_NUM/locations/global/workloadIdentityPools/$PROVIDER_NAME/attribute.repository/$YOUR_ORGANIZATION/$YOUR_REPOSITORY"
```

## GitHub Actionsワークフローの実装
```.github/workflows/cloudrun.yml```に以下のようなワークフローを実装します（```cloudrun.yml```は好きなファイル名で大丈夫です）：
```yaml:.github/workflows/cloudrun.yml
on:
  push:
    branches: [main]  # デプロイ対象のブランチを指定

jobs: 
  prod: 
    runs-on: ubuntu-latest
    permissions: 
      id-token: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Google Auth
        id: auth
        uses: 'google-github-actions/auth@v2'
        with:
          project_id: '$PROJECT_ID'
          workload_identity_provider: 'projects/$PROJECT_NUM/locations/global/workloadIdentityPools/$PROVIDER_NAME/providers/github'
          token_format: 'access_token'
          service_account: '$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com'
      - name: Docker Auth
        id: docker-auth
        uses: 'docker/login-action@v3'
        with:
          username: 'oauth2accesstoken'
          password: '${{ steps.auth.outputs.access_token }}'
          registry: 'asia-northeast1-docker.pkg.dev'
      - name: Build, tag and push container
        id: build-image
        uses: docker/build-push-action@v6
        with:
          context: './src/' # アプリケーションのソースコードが配置されているディレクトリ
          push: true 
          tags: |
             asia-northeast1-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/$IMAGE_NAME:${{ github.sha }}
             asia-northeast1-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/$IMAGE_NAME:latest
      - id: 'deploy'
        uses: 'google-github-actions/deploy-cloudrun@v2'
        with:
          service: '$SERVICE_NAME'
          region: 'asia-northeast1'
          image: 'asia-northeast1-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/$IMAGE_NAME:latest'
```

## おわりに
Google CloudのCloud RunへGithub Actionsを用いた自動デプロイを実装しました。ワークフローを準備して、コマンドベースでシンプルに実装できたのではと思います。ローカル環境から毎回コマンドを叩くよりも一回これらを実装して自動化してあげた方がやっぱり楽かなと思いました。
是非皆さんも実装してみてください。

## 参考文献
- [GitHub Actions で OIDC を使用して Google Cloud へ認証を行う](https://zenn.dev/kou_pg_0131/articles/gh-actions-oidc-gcp)
- [Github Actions + Cloud Deploy でモダンな CI/CD 実装](https://zenn.dev/t_hayashi/articles/bb800299388c1c#%E7%B6%99%E7%B6%9A%E7%9A%84%E3%82%A4%E3%83%B3%E3%83%86%E3%82%B0%E3%83%AC%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%EF%BC%9Agithub-actions)
- [Workload Identity 連携を利用して GitHub Actions から Cloud Run にデプロイ](https://zenn.dev/teasegasugoi/articles/25ff7b339d528e)
