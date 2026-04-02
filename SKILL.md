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

1. **`--no-input` フラグを付与する** — すべてのコマンドにグローバルフラグ `--no-input` を付けることで、対話プロンプトを無効化できる
2. **必須パラメータをすべてフラグで指定する** — 省略すると対話的選択が発生し `interactive selection requires a TTY` エラーになる
3. **`conoha server create` では `--flavor`、`--image`、`--key-name` を必ず指定する**
4. **`conoha app` サブコマンドでは `--app-name` を必ず指定する** — 省略するとアプリ名の入力プロンプトが発生する
5. **破壊的コマンド（delete、destroy、stop）は `--yes` フラグで確認をスキップする**

```bash
# 非TTY環境での例
conoha server create --name my-server --flavor <ID> --image <ID> --key-name <キー名> --no-input --wait
conoha app deploy my-server --app-name myapp --no-input
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

| コマンド | 説明 |
|---------|------|
| `conoha app init <ID\|名前>` | サーバーにDocker環境を初期化する |
| `conoha app deploy <ID\|名前>` | カレントディレクトリをサーバーにデプロイする |
| `conoha app status <ID\|名前>` | アプリのコンテナ状態を表示する |
| `conoha app logs <ID\|名前> --follow` | アプリのログをストリーミング表示する |
| `conoha app stop <ID\|名前>` | アプリのコンテナを停止する |
| `conoha app restart <ID\|名前>` | アプリのコンテナを再起動する |

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
