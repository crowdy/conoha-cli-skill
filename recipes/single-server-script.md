# シングルサーバースクリプトデプロイ

## 概要

カスタムスクリプトをサーバー1台にSSH経由で実行するレシピ。Docker Composeを使わず、任意のセットアップを行う場合に使用する。

## 基本構成

- **ノード数**: 1
- **OS**: Ubuntu
- **方式**: `conoha server deploy` でローカルスクリプトをアップロード・実行

```
[ローカルPC] -- SSH --> [ConoHa VPS]
                         └── スクリプトを実行
                             ├── パッケージインストール
                             ├── 設定ファイル配置
                             └── サービス起動
```

## 手順

### 1. 事前準備

フレーバーとイメージを確認する：

```bash
conoha flavor list
conoha image list
```

### 2. サーバー作成

```bash
conoha server create --name my-server --flavor <フレーバーID> --image <UbuntuイメージID> --key-name my-key --wait
```

### 3. デプロイスクリプトの作成

ローカルにスクリプトファイルを作成する。例：Nginx + Node.jsのセットアップ：

```bash
#!/bin/bash
set -euo pipefail

# パッケージ更新
apt-get update && apt-get upgrade -y

# Nginxインストール
apt-get install -y nginx

# Node.jsインストール（LTS）
curl -fsSL https://deb.nodesource.com/setup_lts.x | bash -
apt-get install -y nodejs

# Nginx設定（リバースプロキシ）
cat > /etc/nginx/sites-available/app <<'NGINX'
server {
    listen 80;
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
NGINX

ln -sf /etc/nginx/sites-available/app /etc/nginx/sites-enabled/default
systemctl restart nginx

echo "Setup complete"
```

### 4. スクリプト実行

```bash
conoha server deploy my-server --script setup.sh
```

環境変数を渡す場合：

```bash
conoha server deploy my-server --script setup.sh --env DB_HOST=localhost --env DB_PORT=5432
```

スクリプト内で環境変数を参照できる：`$DB_HOST`, `$DB_PORT`

### 5. 動作確認

```bash
# SSH接続して状態を確認する
ssh root@<サーバーIP> "systemctl status nginx"
ssh root@<サーバーIP> "curl -s localhost"
```

## カスタマイズ

### データベースセットアップ

PostgreSQLを追加する場合のスクリプト例：

```bash
#!/bin/bash
set -euo pipefail
apt-get update
apt-get install -y postgresql postgresql-contrib
systemctl enable postgresql
sudo -u postgres createuser --superuser $APP_USER
sudo -u postgres createdb $APP_DB -O $APP_USER
```

```bash
conoha server deploy my-server --script setup-db.sh --env APP_USER=myapp --env APP_DB=myapp_db
```

### 複数スクリプトの段階実行

複雑なセットアップは複数スクリプトに分割して実行する：

```bash
conoha server deploy my-server --script 01-base-setup.sh
conoha server deploy my-server --script 02-app-setup.sh --env APP_PORT=3000
conoha server deploy my-server --script 03-nginx-setup.sh --env APP_PORT=3000
```

## トラブルシューティング

| 問題 | 対処 |
|------|------|
| スクリプトが途中で失敗する | `set -euo pipefail` を先頭に入れて失敗箇所を特定する |
| SSH接続できない | サーバーがACTIVE状態か確認する（`conoha server show <名前>`） |
| 環境変数が展開されない | `--env KEY=VALUE` の形式を確認する（`=` の前後にスペースを入れない） |
