---
name: conoha-cli
description: >
  ConoHa VPS3 CLIによるインフラ構築スキル。サーバー作成、アプリデプロイ、
  Kubernetesクラスター、OpenStackプラットフォーム、Slurmクラスターの構築を支援。
  FigmaデザインからWebアプリを生成してデプロイすることも可能。
  「ConoHaでサーバーを作って」「k8sクラスターを構築して」「アプリをデプロイして」
  「Figmaからデプロイ」「デザインからアプリを作って」
  などのリクエストでトリガー。
---

# ConoHa CLI スキル

ConoHa VPS3 CLIを使ったインフラ構築ガイド。

## 前提条件

- `conoha-cli` がインストール済みであること
- `conoha auth login` で認証済みであること
- SSHキーペアが登録済みであること（`conoha keypair create <name>`）

## 非TTY環境（Windows等）での注意事項

Windows（Windows Server 2019等）やCI/CD環境など、TTYが利用できない環境では対話的なプロンプトが動作しない。以下のルールに従うこと：

1. **`--no-input` フラグを付与する** — すべてのコマンドにグローバルフラグ `--no-input` を付けることで、対話プロンプトを無効化できる。CI/CDパイプラインでは環境変数 `CONOHA_NO_INPUT=1` を設定する方が便利な場合がある
2. **必須パラメータをすべてフラグで指定する** — 省略すると対話的プロンプトが発生しエラーになる（例: `interactive selection requires a TTY`）
3. **`conoha server create` では `--flavor`、`--image`、`--key-name` を必ず指定する**
4. **`conoha app` サブコマンドでは `--app-name` を必ず指定する** — 省略するとアプリ名の入力プロンプトが発生する
5. **確認プロンプトが出る破壊的コマンド（`server delete`、`app destroy`、`app stop`）は `--yes` フラグで確認をスキップする**（環境変数: `CONOHA_YES=1`）。`app restart` / `app rollback` は確認プロンプトを出さないため `--yes` は不要 (指定しても no-op)
6. **アプリの既存モードと異なる `--proxy` / `--no-proxy` を指定するとモード不一致エラーで停止する** — 切り替えたい場合は `conoha app destroy --yes` → 反対モードで `init` し直す

```bash
# 非TTY環境での例
conoha server create --name my-server --flavor <ID> --image <ID> --key-name <キー名> --no-input --wait
conoha app deploy my-server --app-name myapp --no-input

# CI/CDパイプラインでは環境変数でまとめて設定可能
export CONOHA_NO_INPUT=1
export CONOHA_YES=1
```

## 基本操作

### サーバー管理

| コマンド | 説明 |
|---------|------|
| `conoha server list` | サーバー一覧を表示する |
| `conoha server create --name <名前> --flavor <ID> --image <ID> --key-name <キー名>` | サーバーを作成する |
| `conoha server create --name <名前> --flavor <ID> --image <ID> --key-name <キー名> --wait` | サーバー作成完了まで待機する |
| `conoha server show <ID\|名前>` | サーバー詳細を表示する |
| `conoha server delete <ID\|名前>` | サーバーを削除する |
| `conoha server deploy <ID\|名前> --script <ファイル> --env KEY=VALUE` | スクリプトをSSH経由で実行する |

### フレーバー・イメージ

| コマンド | 説明 |
|---------|------|
| `conoha flavor list` | 利用可能なフレーバー一覧を表示する |
| `conoha image list` | 利用可能なイメージ一覧を表示する |

### ネットワーク

| コマンド | 説明 |
|---------|------|
| `conoha network list` | ネットワーク一覧を表示する |
| `conoha network create --name <名前>` | プライベートネットワークを作成する |
| `conoha network subnet create --network-id <ID> --cidr <CIDR>` | サブネットを作成する |
| `conoha network sg list` | セキュリティグループ一覧を表示する |
| `conoha network sg create --name <名前>` | セキュリティグループを作成する |
| `conoha network sgr create --security-group-id <ID> --direction ingress --protocol tcp --port-min <ポート> --port-max <ポート> --remote-ip <CIDR>` | セキュリティグループルールを追加する |

### キーペア

| コマンド | 説明 |
|---------|------|
| `conoha keypair list` | キーペア一覧を表示する |
| `conoha keypair create <名前>` | キーペアを作成する |

### アプリデプロイ

`conoha app` は 2 つのデプロイモードを提供する：

- **proxy モード (既定)** — conoha-proxy 経由の blue/green デプロイ。ドメイン + Let's Encrypt TLS、`conoha.yml` 必須、事前に `conoha proxy boot` が必要。
- **no-proxy モード (`--no-proxy`)** — フラット単一スロット。`conoha.yml` / proxy / DNS 不要。テスト・内部 VPS・非 HTTP サービス・ホビー用途に適する。

`conoha app init` がサーバーに `.conoha-mode` マーカーを書き込み、以降の lifecycle コマンドは自動的に同じモードで動作する。`--proxy` / `--no-proxy` フラグはマーカーを上書きするが、不一致ならエラーになる。

| コマンド | 説明 |
|---------|------|
| `conoha app init <server>` | proxy モードで初期化 (conoha.yml と `conoha proxy boot` 済み前提) |
| `conoha app init <server> --app-name <app> --no-proxy` | no-proxy モードで初期化 (Docker / Compose のみインストール) |
| `conoha app deploy <server>` | カレントディレクトリをデプロイ (モードはマーカーから自動判別) |
| `conoha app deploy <server> --slot <id>` | slot ID を固定 (proxy モード) |
| `conoha app rollback <server>` | 前 slot へ即時ロールバック (proxy モードのみ、drain 窓内) |
| `conoha app status <server> --app-name <app>` | コンテナ状態を表示 |
| `conoha app logs <server> --app-name <app> --follow` | ログをストリーミング |
| `conoha app logs <server> --app-name <app> --service <svc>` | 特定サービスのログ |
| `conoha app stop <server> --app-name <app>` | コンテナを停止 |
| `conoha app restart <server> --app-name <app>` | コンテナを再起動 |
| `conoha app destroy <server> --app-name <app> --yes` | アプリをサーバーから完全削除 (非対話) |
| `conoha app list <server>` | サーバー上のデプロイ済みアプリ一覧 |
| `conoha app env set <server> --app-name <app> KEY=VALUE` | サーバー側永続環境変数を設定 |
| `conoha app env list/get/unset` | 環境変数の一覧・取得・削除 |

モード切り替えは `destroy` → 反対モードで `init`。同一 VPS で `<app-name>` が異なれば 2 モードを並列で共存可。

選択指針:
- ドメイン + HTTPS が必要 / 本番公開 → **proxy**
- `docker compose up -d --build` 相当で十分 / DNS 未取得 / 社内・検証・ホビー → **no-proxy**

## レシピ一覧

ユーザーのリクエストに応じて、該当するレシピファイルを読み込んで手順を実行する。

| レシピ | 用途 | ファイル |
|--------|------|---------|
| シングルサーバーアプリ | Docker Composeアプリのデプロイ | [recipes/single-server-app.md](recipes/single-server-app.md) |
| シングルサーバースクリプト | カスタムスクリプトによるセットアップ | [recipes/single-server-script.md](recipes/single-server-script.md) |
| Kubernetesクラスター | k3sによるマルチノードk8sクラスター | [recipes/k8s-cluster.md](recipes/k8s-cluster.md) |
| OpenStackプラットフォーム | DevStackによるOpenStack環境 | [recipes/openstack-platform.md](recipes/openstack-platform.md) |
| Slurmクラスター | HPCジョブスケジューラクラスター | [recipes/slurm-cluster.md](recipes/slurm-cluster.md) |
| FigmaデザインからWebアプリ | FigmaデザインからReactコード生成・デプロイ | [recipes/figma-to-deploy.md](recipes/figma-to-deploy.md) |

## 共通パターン

### マルチサーバー作成

複数サーバーを作成する場合、命名規則を統一する：

```bash
# 例: k8sクラスター
conoha server create --name k8s-master-1 --flavor <ID> --image <ID> --key-name <キー名> --security-group <SG名> --wait
conoha server create --name k8s-worker-1 --flavor <ID> --image <ID> --key-name <キー名> --security-group <SG名> --wait
conoha server create --name k8s-worker-2 --flavor <ID> --image <ID> --key-name <キー名> --security-group <SG名> --wait
```

### セキュリティグループの事前準備

クラスター構築前に必要なポートを開放するセキュリティグループを作成する：

```bash
# セキュリティグループ作成
conoha network sg create --name <クラスター名>-sg

# 必要なルールを追加（例: SSH + クラスター固有ポート）
conoha network sgr create --security-group-id <SG-ID> --direction ingress --protocol tcp --port-min 22 --port-max 22 --remote-ip 0.0.0.0/0
```

### デプロイスクリプト実行順序

マルチサーバー構成では以下の順序で実行する：

1. 全ノード共通の初期設定スクリプトを実行する
2. マスター/コントローラーノードのセットアップスクリプトを実行する
3. マスターからジョイン情報（トークン等）を取得する
4. ワーカー/コンピュートノードのセットアップスクリプトを実行する（ジョイン情報を`--env`で渡す）

```bash
# 1. 全ノード共通
conoha server deploy <各ノード> --script common-setup.sh

# 2. マスターセットアップ
conoha server deploy <マスター> --script master-setup.sh

# 3. ジョイン情報取得（SSH経由）
TOKEN=$(ssh root@<マスターIP> "取得コマンド")

# 4. ワーカーセットアップ
conoha server deploy <ワーカー> --script worker-setup.sh --env JOIN_TOKEN=$TOKEN
```

## 問題報告

conoha-cli の使用中に異常な動作や予期しないエラーを発見した場合、`gh` コマンドが利用可能であれば GitHub Issue を自動登録する：

```bash
gh issue create --repo crowdy/conoha-cli --title "<エラーの要約>" --body "<再現手順と詳細>"
```

- 再現手順、実行したコマンド、エラーメッセージを含める
- 可能であれば conoha-cli のバージョン（`conoha version`）も記載する
