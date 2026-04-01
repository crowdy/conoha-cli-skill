# シングルサーバーアプリデプロイ

## 概要

Docker Composeアプリをサーバー1台にデプロイするレシピ。`conoha app` コマンドを使用し、カレントディレクトリのDocker Composeプロジェクトをサーバーに転送・起動する。

## 基本構成

- **ノード数**: 1
- **OS**: Ubuntu
- **必須**: Docker Compose対応の `docker-compose.yml` がカレントディレクトリにあること

```
[ローカルPC] -- tar+SSH --> [ConoHa VPS]
                             └── /opt/conoha/<app-name>/
                                 ├── docker-compose.yml
                                 ├── Dockerfile
                                 ├── .env (← .env.server からコピー)
                                 └── (ソースコード)
```

## 手順

### 1. 事前準備

フレーバーとイメージを確認する：

```bash
conoha flavor list
conoha image list
```

キーペアが未作成の場合は作成する：

```bash
conoha keypair create my-key
```

### 2. サーバー作成

```bash
conoha server create --name my-app-server --flavor <フレーバーID> --image <UbuntuイメージID> --key-name my-key --wait
```

`--wait` を付けてサーバーがACTIVEになるまで待機する。

### 3. アプリ初期化

サーバーにDocker環境をセットアップする：

```bash
conoha app init my-app-server
```

これにより以下が実行される：
- Docker/Docker Composeのインストール
- git bare リポジトリの作成（post-receive hook付き）

### 4. アプリデプロイ

カレントディレクトリ（`docker-compose.yml` があるディレクトリ）で実行する：

```bash
conoha app deploy my-app-server
```

これにより以下が実行される：
- カレントディレクトリのtar.gzアーカイブ作成（`.git/` 除外）
- SSH経由でアップロード
- `/opt/conoha/<app-name>/` に展開
- `.env.server` → `.env` コピー（存在する場合）
- `docker compose up -d --build --remove-orphans` 実行

### 5. 動作確認

```bash
# コンテナ状態を確認する
conoha app status my-app-server

# ログを確認する
conoha app logs my-app-server --follow

# 直接SSHでアクセスする場合
ssh root@<サーバーIP> "curl -s localhost:<ポート>"
```

## カスタマイズ

### 環境変数

サーバー側に永続的な環境変数を設定する（デプロイを跨いで維持される）：

```bash
conoha app env set my-app-server DATABASE_URL=postgres://...
conoha app env list my-app-server
```

または、ローカルに `.env.server` ファイルを作成してデプロイ時に自動コピーさせる。

### アプリの管理

```bash
# 停止
conoha app stop my-app-server

# 再起動
conoha app restart my-app-server

# ログ（特定サービス）
conoha app logs my-app-server --service web
```

## トラブルシューティング

| 問題 | 対処 |
|------|------|
| `docker compose up` が失敗する | `conoha app logs my-app-server` でエラーを確認する |
| ポートにアクセスできない | セキュリティグループで該当ポートが開放されているか確認する |
| デプロイが遅い | `.dockerignore` でnode_modules等を除外する |
