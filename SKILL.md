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
3. **`conoha server create` では `--flavor`、`--image`、`--key-name`、`--security-group` を必ず指定する** — `--security-group` は repeatable。最低限 SSH 到達可能な SG を含めること。一般的には `default` + `IPv4v6-SSH` (proxy モードを使うなら `IPv4v6-Web` も追加)。`default` 単独では SSH 22 が届かないケースが多い。SG を省略すると `--no-input` 下で `selection required but --no-input is set` または silent fail (exit 0) になる ([crowdy/conoha-cli#155](https://github.com/crowdy/conoha-cli/issues/155))
4. **`conoha app` サブコマンドでは `--app-name` を必ず指定する** — 省略するとアプリ名の入力プロンプトが発生する。値は **DNS-1123 ラベル** (小文字英数とハイフン、英数で開始/終了、`^[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?$`, 1–63 文字) とすること。例: `my-app` ✅ / `my_app` ❌ / `MyApp` ❌<br>※ 現行 CLI は `app init/deploy` 時に DNS-1123 で値を弾く ([crowdy/conoha-cli#124](https://github.com/crowdy/conoha-cli/pull/124))。旧バージョンで作成された underscore/大文字を含むアプリは `app destroy` で `/opt/conoha/<name>` の作業ディレクトリが残留する既知の不具合があったため ([crowdy/conoha-cli#119](https://github.com/crowdy/conoha-cli/issues/119))、レガシー名のアプリを片付ける場合のみ `app destroy` 後に手動で同パスを確認・削除する
5. **確認プロンプトが出るコマンドには `--yes` フラグで確認をスキップする**（環境変数: `CONOHA_YES=1`）。対象は破壊的コマンド (`server delete`、`app destroy`、`app stop`) に加えて **`server create` 自体** — boot volume を新規作成する確認プロンプトが出るため `--yes` がないと `confirmation required but --no-input is set; use --yes to auto-confirm` で失敗する (この時 boot volume は orphan として残るので注意、後述の「在庫切れリトライ」参照)。`app restart` / `app rollback` は確認プロンプトを出さないため `--yes` は不要 (指定しても no-op)
6. **アプリの既存モードと異なる `--proxy` / `--no-proxy` を指定するとモード不一致エラーで停止する** — 切り替えたい場合は `conoha app destroy --yes` → 反対モードで `init` し直す

```bash
# 非TTY環境での例
conoha server create --name my-server --flavor <ID> --image <ID> --key-name <キー名> \
  --security-group default --security-group IPv4v6-SSH \
  --no-input --yes --wait
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
| `conoha server create --name <名前> --flavor <ID> --image <ID> --key-name <キー名> --security-group <SG名>` | サーバーを作成する (`--security-group` は repeatable、SSH 用 SG を必ず含める) |
| `conoha server create ... --volume <既存ボリュームID>` | 既存 boot volume を再利用してサーバーを作成する (`--image` も CLI 上必須なので作成元と同じ ID を渡す) |
| `conoha server create ... --wait` | サーバー作成完了まで待機する |
| `conoha server show <ID\|名前>` | サーバー詳細を表示する |
| `conoha server delete <ID\|名前> --yes` | サーバーを削除する (**boot volume は残る** — `volume list` で確認し個別に削除するか、下記 `--delete-boot-volume` を使う) |
| `conoha server delete <ID\|名前> --delete-boot-volume --yes --wait` | サーバーと boot volume を一括削除する (推奨。残し忘れによる quota 圧迫を防ぐ) |
| `conoha server add-security-group <ID\|名前> --name <SG名> --yes` (alias `add-sg`) | 既存サーバーに SG を後付けする (SSH 不通時の救済に有効) |
| `conoha server remove-security-group <ID\|名前> --name <SG名> --yes` (alias `remove-sg`) | サーバーから SG を取り外す |
| `conoha server deploy <ID\|名前> --script <ファイル> --env KEY=VALUE` | スクリプトをSSH経由で実行する |

### フレーバー・イメージ

| コマンド | 説明 |
|---------|------|
| `conoha flavor list` | 利用可能なフレーバー一覧を表示する |
| `conoha image list` | 利用可能なイメージ一覧を表示する |

### ボリューム

| コマンド | 説明 |
|---------|------|
| `conoha volume list` | ボリューム一覧を表示する (orphan の boot volume 確認に使う) |
| `conoha volume show <ID>` | ボリューム詳細を表示する |
| `conoha volume delete <ID> --yes` | ボリュームを削除する (server delete 時に残った boot volume の救済) |

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
| `conoha app init <server> --app-name <app> --no-proxy` | no-proxy モードで初期化 (Docker / Compose が入っていることを検証。インストールはしない) |
| `conoha app deploy <server>` | カレントディレクトリをデプロイ (モードはマーカーから自動判別) |
| `conoha app deploy <server> --slot <id>` | slot ID を固定 (proxy モード) |
| `conoha app rollback <server>` | 前 slot へ即時ロールバック (proxy モードのみ、drain 窓内) |
| `conoha app status <server> --app-name <app>` | コンテナ状態を表示 |
| `conoha app logs <server> --app-name <app> --follow` | ログをストリーミング |
| `conoha app logs <server> --app-name <app> --service <svc>` | 特定サービスのログ |
| `conoha app stop <server> --app-name <app>` | コンテナを停止 |
| `conoha app restart <server> --app-name <app>` | コンテナを再起動 |
| `conoha app destroy <server> --app-name <app> --yes` | アプリをサーバーから完全削除 (非対話) |
| `conoha app reset <server> --app-name <app> --yes` | `destroy` + `init` + `deploy` を 1 SSH セッションで連続実行 (再デプロイ用、v0.5.6 以降)。`app env set` で投入した値は `destroy` で消えるため、必要なら直前に export して再投入する |
| `conoha app list <server>` | サーバー上のデプロイ済みアプリ一覧 |
| `conoha app env set <server> --app-name <app> KEY=VALUE` | サーバー側永続環境変数を設定 |
| `conoha app env list/get/unset` | 環境変数の一覧・取得・削除 |

> **注意**: `app env` はサーバー側 `/opt/conoha/<app>.env.server` に書き込むが、**現状その値が実際にコンテナに反映されるのは no-proxy モードのデプロイ時のみ**。proxy モードでは `warning: app env has no effect on proxy-mode deployed slots; see #94 for the redesign` が出る ([crowdy/conoha-cli#94](https://github.com/crowdy/conoha-cli/issues/94) 予定)。proxy モードの環境変数は当面 compose 側の `environment:` / `env_file:` で渡す。

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
| リリーススモーク | post-merge / pre-tag の e2e 検証 (実 VPS) | [recipes/release-smoke.md](recipes/release-smoke.md) |

## イメージ別 known-issues

ConoHa の各イメージには CLI が知っておくべきデフォルト挙動の差がある。同じコマンドでもイメージ次第で挙動が変わるので、選んだ image_id ごとに以下を確認する。

| イメージ | 注意点 | 影響 |
|---|---|---|
| `vmi-docker-29.2-ubuntu-24.04-amd64` | UFW が `policy DROP` で SSH のみ allow / kernel default `net.ipv4.ip_unprivileged_port_start=1024` | `proxy boot` が `bind: permission denied` で crash-loop / 外部から 80,443 が unreachable。CLI v0.6.x 以降は BootScript が両方を自動修正するが、旧版や独自イメージでは手動対応が必要 |

新しいイメージで動作させる時は最初に **ad-hoc な実 VPS スモーク** で UFW / sysctl / TOFU のクラスを確認すること。スモーク手順は [recipes/release-smoke.md](recipes/release-smoke.md) に codify されている。

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

### SSH 到達性を担保する SG attach パターン

`conoha server create --security-group default` だけで作成すると `default` SG のインバウンドが極めて制限的なため SSH (22) すら届かないケースが頻発する。**作成時に必要な SG をすべて attach** するのが基本：

```bash
conoha server create --name <名前> --flavor <ID> --image <ID> --key-name <キー名> \
  --security-group default \
  --security-group IPv4v6-SSH \
  --security-group IPv4v6-Web \
  --no-input --yes --wait
```

既に作成済みで SSH が届かない場合は **VPS を消さずに後付け** で復旧できる：

```bash
conoha server add-sg <ID|名前> --name IPv4v6-SSH --yes
conoha server add-sg <ID|名前> --name IPv4v6-Web --yes
```

### 在庫切れリトライと既存ボリュームの再利用

`conoha server create` は ConoHa3 側で「No compute resource in stock」となった場合に HTTP 500 を返す (特に L4 GPU フレーバーで発生しやすい)。この API は**サーバー作成前に boot volume を先に作る仕様**のため、失敗時には boot volume だけが orphan として残る：

```
boot volume 3a156cca-... was created but server creation failed.
You can delete it with: conoha volume delete 3a156cca-...
API error (HTTP 500): {"code": 500, "error": "No compute resource in stock."}
```

このボリュームは捨てずに `--volume <既存ボリュームID>` で次のリトライに渡せる：

```bash
# 在庫が戻ったら、同じ image ID と組み合わせて再試行する
conoha server create \
  --name <名前> \
  --flavor <ID> \
  --image <作成元と同じ image ID> \
  --volume <既存ボリュームID> \
  --key-name <キー名> \
  --security-group default --security-group IPv4v6-SSH \
  --no-input --yes --wait
```

ポイント:

- リトライ毎に新規ボリュームを作る必要がなく、orphan のクリーンアップも不要
- 同じフレーバーが在庫切れでも、別のフレーバー (例: より大きいプラン) で同じボリュームを再利用してサーバー作成できる場合がある
- 不要になった orphan は `conoha volume delete <ID> --yes` で個別に削除する
- L4 flavor は GPU 専用ネットワーク (`ext-gpu-v4v6-*`) に自動割り当てされるため `--network` 指定は不要

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
