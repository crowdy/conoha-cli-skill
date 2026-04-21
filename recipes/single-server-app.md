# シングルサーバーアプリデプロイ

## 概要

Docker Compose アプリをサーバー 1 台にデプロイするレシピ。`conoha app` コマンドでカレントディレクトリの Compose プロジェクトを転送・起動する。

`conoha app` は 2 つのモードを提供するので、用途に応じて選ぶ:

| モード | いつ選ぶか | 事前準備 |
|---|---|---|
| **proxy (blue/green, 既定)** | ドメイン + HTTPS が必要、本番公開、無停止デプロイ | `conoha.yml`、`conoha proxy boot`、DNS A レコード |
| **no-proxy (flat)** | テスト、社内ツール、非 HTTP サービス、ホビー用途 | `docker-compose.yml` だけ |

## 共通の前提

- `conoha auth login` で認証済み
- キーペアが登録済み (`conoha keypair create <name>`)
- カレントディレクトリに `docker-compose.yml` (または `compose.yml`) がある
- 非 TTY 環境では `--no-input` / `--yes` / 必須フラグを明示 (SKILL.md 冒頭の注意を参照)

### サーバー作成 (両モード共通)

```bash
conoha flavor list
conoha image list
conoha keypair create my-key    # 既存ならスキップ

conoha server create \
  --name my-app-server \
  --flavor <フレーバーID> \
  --image <UbuntuイメージID> \
  --key-name my-key \
  --wait
```

`--wait` でサーバーが ACTIVE になるまで待つ。

---

## モード A: proxy blue/green (推奨)

### 1. `conoha.yml` をレポジトリルートに作成

```yaml
name: myapp
hosts:
  - app.example.com
web:
  service: web    # docker-compose.yml のサービス名と一致
  port: 8080      # コンテナ側 listen ポート
# 任意:
# compose_file: docker-compose.yml
# accessories: [db, redis]
# health: { path: /healthz, interval_ms: 1000, timeout_ms: 500, healthy_threshold: 2, unhealthy_threshold: 3 }
# deploy: { drain_ms: 5000 }
```

スキーマ詳細は conoha-cli README を参照。

### 2. DNS A レコードを VPS に向ける

Let's Encrypt HTTP-01 検証に必要。ACME 発行前にポート 80 が到達できる状態にしておく。

### 3. conoha-proxy をブート

```bash
conoha proxy boot my-app-server --acme-email ops@example.com
```

### 4. アプリを proxy に登録

```bash
conoha app init my-app-server
```

これで `.conoha-mode=proxy` のマーカーが書き込まれ、proxy に service が登録される。proxy モードではアプリ名は `conoha.yml` の `name:` フィールドから取得するので `--app-name` は不要 (指定しても無視される)。

### 5. デプロイ

カレントディレクトリで実行:

```bash
conoha app deploy my-app-server
```

実行内容:
- カレントディレクトリを tar.gz アーカイブ化 (`.git/` 除外)
- SSH で転送
- `/opt/conoha/myapp/<slot>/` に展開 (slot は git short SHA または timestamp、`--slot <id>` で上書き可)
- 新 slot を動的ポートで起動
- proxy が health probe → 新 slot に swap
- drain 窓経過後に旧 slot をテアダウン

### 6. 動作確認 / 管理

```bash
conoha app status my-app-server --app-name myapp
conoha app logs my-app-server --app-name myapp --follow
```

ロールバック (drain 窓内のみ):

```bash
conoha app rollback my-app-server
```

廃棄:

```bash
conoha app destroy my-app-server --app-name myapp --yes
```

---

## モード B: no-proxy (flat)

proxy / DNS / TLS が不要なとき。`docker-compose.yml` があれば動く最短経路。

### 1. 初期化

```bash
conoha app init my-app-server --app-name myapp --no-proxy
```

これで `.conoha-mode=no-proxy` のマーカーが書き込まれる。Docker / Compose がインストールされるが、proxy コンテナは起動しない。

### 2. デプロイ

```bash
conoha app deploy my-app-server --app-name myapp --no-proxy
```

実行内容:
- カレントディレクトリを tar.gz アーカイブ化 (`.git/` 除外)
- `/opt/conoha/myapp/` に展開 (フラット、slot なし)
- サーバー側 `/opt/conoha/<app>.env.server` (`conoha app env set` で登録) があれば、リポジトリの `.env` に追記する形で合成
- `docker compose up -d --build --remove-orphans`

### 3. 動作確認 / 管理

以降のコマンドはマーカーから自動判別されるので `--no-proxy` の再指定は不要 (付けても動く):

```bash
conoha app status my-app-server --app-name myapp
conoha app logs my-app-server --app-name myapp --follow
conoha app logs my-app-server --app-name myapp --service web
conoha app stop my-app-server --app-name myapp --yes
conoha app restart my-app-server --app-name myapp --yes
conoha app destroy my-app-server --app-name myapp --yes
```

ポート公開は `docker-compose.yml` の `ports` セクションで直接行う。セキュリティグループで該当ポートを開放すること。

no-proxy モードには blue/green swap が無いため `rollback` は使えない (実行すると `rollback is not supported in no-proxy mode` エラーが出る)。前のリビジョンに戻すには `git checkout <sha> && conoha app deploy --no-proxy --app-name myapp my-app-server` で再デプロイする。

---

## 環境変数

サーバー側に永続する環境変数はデプロイを跨いで維持される (両モード共通):

```bash
conoha app env set my-app-server --app-name myapp DATABASE_URL=postgres://...
conoha app env list my-app-server --app-name myapp
conoha app env get my-app-server --app-name myapp DATABASE_URL
conoha app env unset my-app-server --app-name myapp DATABASE_URL
```

デプロイ時の `.env` 合成は **リポジトリの `.env` → サーバー側の `/opt/conoha/<app>.env.server` を追記** の順で行われるため、`conoha app env set` で登録した値が後勝ちで上書きする。リポジトリ側にコミットした `.env` があればそれも `docker compose` に渡される。

## モード切り替え

既存アプリのモードを変えるときは、一度破棄してから反対モードで再 init する:

```bash
conoha app destroy my-app-server --app-name myapp --yes
conoha app init my-app-server --app-name myapp --no-proxy   # または --proxy (フラグ省略)
```

同一 VPS 上で `<app-name>` が異なれば、proxy / no-proxy を並列に共存可能。

## トラブルシューティング

| 問題 | 対処 |
|------|------|
| `mode conflict` エラー | `--proxy` / `--no-proxy` フラグが既存マーカーと不一致。上記「モード切り替え」を参照 |
| `docker compose up` が失敗 | `conoha app logs` でエラー確認 |
| ポートにアクセス不可 (no-proxy) | セキュリティグループで `docker-compose.yml` の `ports` が開放されているか確認 |
| Let's Encrypt 発行失敗 (proxy) | DNS A レコードが VPS を指しているか、ポート 80 が到達可能かを確認 |
| デプロイが遅い | `.dockerignore` で `node_modules` 等を除外 |
