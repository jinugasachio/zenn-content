---
title: "Github Actions から OIDC で Google Container Registry にイメージをプッシュする"
emoji: "🔒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GCP", "GithubActions", "Docker"]
published: false
---

この記事は[株式会社Gincoのテックブログ](https://tech.ginco.co.jp/)として書いています。

# はじめに
- 業務で掲題のワークフローを実装する必要があったのでまとめました
- OIDC を利用することで Github にクレデンシャルを渡す必要がなくなり、よりセキュアな運用が可能になります
- コンソールから行う OIDC の設定手順と実装したワークフローが書いてあります 

# 注意事項
- Google Container Registry は [2024年5月15日以降にサポートが終了します](https://cloud.google.com/container-registry/docs/deprecations/container-registry-deprecation?hl=ja)
- 特別な事情がない限りは [Artifact Registry](https://cloud.google.com/artifact-registry/docs/overview?hl=ja) を代わりに使用してください
- `最終的なワークフロー` にある `docker/login-action@v3` の `registry` に渡す値を任意の Artifact Registryのホストに変えればそのまま転用できると思います

# OIDC の設定手順
## 1. Workload Identity のプールとプロバイダを作成
- `IAMと管理` > `Workload Identity 連携` をクリック
![](/images/ga_oidc_to_gcp/tejyun1.png)

- `プールを作成` にアクセス。任意の名前を入力し、`続行` をクリック
![](/images/ga_oidc_to_gcp/tejyun2.png)

- `OpenID Connect（OIDC）` を選択
- 任意のプロバイダ名を入力
- `発行元（URL）`には `https://token.actions.githubusercontent.com` を入力
- `デフォルトのオーディエンス`を選択
![](/images/ga_oidc_to_gcp/tejyun3.png)

- 以下のように属性をマッピングし、`保存`をクリック
  - `google.subject`       => `assertion.sub`
  - `attribute.repository` => `assertion.repository`
  - `attribute.actor`      =>	`assertion.actor`
![](/images/ga_oidc_to_gcp/tejyun4.png)

## 2. Github Actions から利用するサービスアカウントの作成
- `IAMと管理` > `サービスアカウント` をクリック
![](/images/ga_oidc_to_gcp/tejyun5.png)

- `サービスアカウントを作成`をクリック
![](/images/ga_oidc_to_gcp/tejyun6.png)

- 任意の`サービスアカウント名`を入力し、`作成して続行`をクリック
![](/images/ga_oidc_to_gcp/tejyun7.png)

- サービスアカウントに`ストレージ管理者`のロールを付与し、`完了` をクリック
![](/images/ga_oidc_to_gcp/tejyun8.png)



## 3. Github Actions にサービスアカウントの権限借用を許可する
- `IAMと管理` > `サービスアカウント` をクリック
![](/images/ga_oidc_to_gcp/tejyun5.png)

- 作成したサービスアカウントをクリック
![](/images/ga_oidc_to_gcp/tejyun10.png)

- `権限`をクリック
![](/images/ga_oidc_to_gcp/tejyun11.png)

- `アクセス権を付与`をクリック
![](/images/ga_oidc_to_gcp/tejyun12.png)

- 以下のフォーマットで`新しいプリンシパル`を入力、`ロール`は `Workload Identity ユーザー`を選択
  ```
  principalSet://iam.googleapis.com/projects/{GCPプロジェクトのID}/locations/global/workloadIdentityPools/{作成したプールのID}/attribute.repository/{Githubのユーザー名}/{リポジトリ名}
  ```
![](/images/ga_oidc_to_gcp/tejyun13.png)

- `プールの詳細`画面にて`接続済サービスアカウント`に作成したサービスアカウントが表示されていれば OK
  - 反映まで数秒時差があります
  - 表示されない場合、`新しいプリンシパル`の値が間違っている可能性があります
![](/images/ga_oidc_to_gcp/tejyun14.png)

# ワークフローの実装
## ポイントと解説
- `main` ブランチに `v` で始まるタグがプッシュされるとワークフローが実行される
- OIDC 利用には `permissions` に `id-token: write` が必要
- 下記の`最終的なワークフロー`は本番環境用だが [reusable workflow](https://docs.github.com/en/actions/using-workflows/reusing-workflows) と [composite action](https://docs.github.com/ja/actions/creating-actions/creating-a-composite-action) を利用し他環境用にも再利用できるようにている
- `docker/setup-buildx-action@v2` で `docker/build-push-action@v4` においてキャッシュを使えるようにする
- OIDC トークンの発行は `google-github-actions/auth@v1` で行われている
  - `OIDC の設定手順`で作成した workload identity provider のプリンシパルを指定
  - `OIDC の設定手順`で作成した service account のアドレスを指定
- `docker/login-action@v2` でOIDC トークンを使って GCR へ認証
- `docker/metadata-action@v4` でイメージのタグを出力
  - `type=semver,pattern={{raw}}` で 例えば Github 上で `v1.2` というタグがプッシュされた時に同じタグを出力する
  - `type=sha,format=short` で git の short sha をタグとして出力する
- `docker/build-push-action@v4` でイメージのビルドとプッシュ
  - 2023年9月時点で [Experimental となっている](https://docs.docker.com/build/ci/github-actions/cache/)が `type=gha` として Github Action のキャッシュを利用
  - `mode=max` として全ての中間レイヤーもキャッシュさせる。デフォルトは `min` であり最終イメージのレイヤーしかキャッシュされない
  - `provenance: false` [こちらの問題](https://github.com/docker/build-push-action/issues/767)を防ぐために指定が必要でした


## 最終的なワークフロー

```yml:.github/workflows/build-push-prd.yml
name: Call build-push workflow for prd

on:
  push:
    tags:
      - v**

jobs:
  call_build_push_workflow:
    if: ${{ github.ref_name == 'main' }}
    name: Call workflow
    uses: ./.github/workflows/reusable-build-push.yml
    with:
      build_context: path/to/build_context
      file: path/to/Dockerfile
      gcr_host: asia.gcr.io
      gcr_repository_host: asia.gcr.io/your-repository/your-image
    secrets:
      workload_identity_provider: ${{ secrets.WORKLOAD_IDENTITY_PROVIDER_FOR_PRD }}
      service_account_address: ${{ secrets.SERVICE_ACCOUNT_ADDRESS_FOR_PRD }}
```

```yml:.github/workflows/reusable-build-push.yml
name: Reusable build-push workflow 

on:
  workflow_call:
    inputs:
      build_context:
        type: string
        required: true
      build_target:
        type: string
        required: true
      file:
        type: string
        required: true
      push:
        type: boolean
        default: true
        required: false
      gcr_host:
        type: string
        required: true
      gcr_repository_host:
        type: string
        required: true
    secrets:
      workload_identity_provider:
        required: true
      service_account_address:
        required: true

jobs:
  build_push:
    name: build-push image
    runs-on: ubuntu-latest
    permissions:
      contents: 'read'
      id-token: 'write'
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Build-Push image
        uses: ./.github/actions/build-push
        with:
          workload_identity_provider: ${{ secrets.workload_identity_provider }}
          service_account_address: ${{ secrets.service_account_address }}
          gcr_host: ${{ inputs.gcr_host }}
          gcr_repository_host: ${{ inputs.gcr_repository_host }}
          build_context: ${{ inputs.build_context }}
          build_target: ${{ inputs.build_target }}
          file: ${{ inputs.file }}
```

```yml:.github/actions/build-push/action.yml
name: Build-Push
description: Build and push image or only build image
inputs:
  workload_identity_provider:
    description: Principal of identity provider for OIDC
    required: true
  service_account_address:
    description: Service account address for GCP 
    required: true
  gcr_host:
    description: Host of GCR
    required: true
  gcr_repository_host:
    description: Repository host of GCR
    required: true
  build_context:
    description: Build context
    required: false
    default: .
  build_target:
    description: Target to build
    required: true
  file:
    description: File path to Dockerfile
    required: true
  push:
    description: Whether push image or not(= only build)
    required: false
    default: 'true'

runs:
  using: composite
  steps:
    - name: Set up buildx
      uses: docker/setup-buildx-action@v2

    - name: Authenticate to Google Cloud
      id: auth
      uses: 'google-github-actions/auth@v1'
      with:
        token_format: access_token
        workload_identity_provider: ${{ inputs.workload_identity_provider }}
        service_account: ${{ inputs.service_account_address }}

    - name: Login to GCR
      uses: docker/login-action@v2
      with:
        registry: ${{ inputs.gcr_host }}
        username: oauth2accesstoken
        password: ${{ steps.auth.outputs.access_token }}

    - name: Set docker metadata
      id: metadata
      uses: docker/metadata-action@v4
      with:
        images: ${{ inputs.gcr_repository_host}}
        tags: |
          type=semver,pattern={{raw}}
          type=sha,format=short

    - name: Build-Push image
      uses: docker/build-push-action@v4
      with:
        context: ${{ inputs.build_context }}
        target: ${{ inputs.build_target }}
        file: ${{ inputs.file }}
        push: ${{ inputs.push }}
        tags: ${{ steps.metadata.outputs.tags }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        provenance: false
```


# 所感
- CI からクラウドへの認証は OIDC の利用を当たり前としていきたい
- コンソールから都度設定するのは手間なので、社内で統一していくには[こういった Terraform module](https://github.com/terraform-google-modules/terraform-google-github-actions-runners/tree/master/modules/gh-oidc) を利用 or 作成したい

# 最後に
- 株式会社 Gincoではブロックチェーンを学びたい方、ウォレットについて詳しくなりたい方を募集していますので下記リンクから是非ご応募ください。
- [株式会社Ginco の求人一覧](https://herp.careers/v1/ginco)

# 参考
https://cloud.google.com/blog/ja/products/identity-security/enabling-keyless-authentication-from-github-actions
https://cloud.google.com/iam/docs/configuring-workload-identity-federation?hl=ja#github-actions
https://cloud.google.com/iam/docs/workload-identity-federation-with-other-providers?hl=ja
https://docs.github.com/ja/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-google-cloud-platform
