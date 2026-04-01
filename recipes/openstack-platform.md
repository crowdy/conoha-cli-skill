# OpenStackプラットフォーム構築

## 概要

ConoHa VPS上にDevStackを使ったOpenStack環境を構築するレシピ。まずオールインワン（1台）構成で構築し、必要に応じてマルチノードに拡張する。

## 基本構成

- **ノード数**: 1（オールインワン）
- **OS**: Ubuntu
- **OpenStackディストリビューション**: DevStack
- **推奨フレーバー**: 4vCPU / 8GB RAM 以上（OpenStackは多くのリソースを消費する）

```
┌──────────────────────────────┐
│        openstack-aio         │
│  ┌────────┐  ┌────────────┐  │
│  │Keystone│  │  Horizon   │  │
│  ├────────┤  ├────────────┤  │
│  │  Nova  │  │  Neutron   │  │
│  ├────────┤  ├────────────┤  │
│  │ Glance │  │  Cinder    │  │
│  └────────┘  └────────────┘  │
└──────────────────────────────┘
```

## 手順

### 1. 事前準備

セキュリティグループを作成する：

```bash
conoha network sg create --name openstack-sg

# SSH
conoha network sgr create --security-group-id <SG-ID> --direction ingress --protocol tcp --port-min 22 --port-max 22 --remote-ip 0.0.0.0/0

# Horizon (Dashboard)
conoha network sgr create --security-group-id <SG-ID> --direction ingress --protocol tcp --port-min 80 --port-max 80 --remote-ip 0.0.0.0/0

# Keystone (Identity API)
conoha network sgr create --security-group-id <SG-ID> --direction ingress --protocol tcp --port-min 5000 --port-max 5000 --remote-ip 0.0.0.0/0

# Nova (Compute API)
conoha network sgr create --security-group-id <SG-ID> --direction ingress --protocol tcp --port-min 8774 --port-max 8774 --remote-ip 0.0.0.0/0

# VNC Console
conoha network sgr create --security-group-id <SG-ID> --direction ingress --protocol tcp --port-min 6080 --port-max 6080 --remote-ip 0.0.0.0/0
```

### 2. サーバー作成

大きなフレーバーを選択する（8GB RAM以上推奨）：

```bash
conoha flavor list
conoha server create --name openstack-aio --flavor <大型フレーバーID> --image <UbuntuイメージID> --key-name my-key --security-group openstack-sg --wait
```

### 3. DevStackインストール

セットアップスクリプト `openstack-setup.sh` を作成する：

```bash
#!/bin/bash
set -euo pipefail

# stackユーザー作成
useradd -s /bin/bash -d /opt/stack -m stack
chmod +x /opt/stack
echo "stack ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/stack

# DevStackクローン
su - stack -c "git clone https://opendev.org/openstack/devstack /opt/stack/devstack"

# local.conf作成
cat > /opt/stack/devstack/local.conf <<EOF
[[local|localrc]]
ADMIN_PASSWORD=$ADMIN_PASSWORD
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD

HOST_IP=$HOST_IP

# 有効化するサービス
enable_service placement-api
enable_service n-cpu
enable_service n-api
enable_service n-cond
enable_service n-sch
enable_service g-api
enable_service c-sch
enable_service c-api
enable_service c-vol

# Neutron
enable_service q-svc
enable_service q-agt
enable_service q-dhcp
enable_service q-l3
enable_service q-meta

# Horizon
enable_service horizon

LOGFILE=/opt/stack/logs/stack.sh.log
LOGDAYS=1
EOF

chown stack:stack /opt/stack/devstack/local.conf

# DevStack実行（時間がかかる: 20〜40分）
su - stack -c "cd /opt/stack/devstack && ./stack.sh"

echo "DevStack installation complete"
```

実行する：

```bash
HOST_IP=$(conoha server show openstack-aio -o json | jq -r '.addresses | to_entries[0].value[] | select(.version == 4) | .addr')
conoha server deploy openstack-aio --script openstack-setup.sh --env ADMIN_PASSWORD=SecurePass123 --env HOST_IP=$HOST_IP
```

注意: DevStackのインストールには20〜40分かかる。`conoha server deploy` のタイムアウトに注意する。

### 4. 動作確認

```bash
# Horizonダッシュボードにアクセスする
echo "http://$HOST_IP/dashboard"
# ユーザー: admin, パスワード: 上で設定したADMIN_PASSWORD

# OpenStack CLIで確認する
ssh root@$HOST_IP "su - stack -c 'source /opt/stack/devstack/openrc admin admin && openstack service list'"
```

## カスタマイズ

### マルチノード構成

コントローラー1台 + コンピュート2台に拡張する場合：

1. 追加のサーバーを作成する：

```bash
conoha server create --name openstack-compute-1 --flavor <フレーバーID> --image <UbuntuイメージID> --key-name my-key --security-group openstack-sg --wait
conoha server create --name openstack-compute-2 --flavor <フレーバーID> --image <UbuntuイメージID> --key-name my-key --security-group openstack-sg --wait
```

2. コンピュートノード用の `local.conf` でコントローラーのIPを `SERVICE_HOST` に設定する。

### 有効化サービスの変更

`local.conf` の `enable_service` / `disable_service` で調整する。例：Swiftを追加する場合：

```bash
enable_service s-proxy s-object s-container s-account
```

## トラブルシューティング

| 問題 | 対処 |
|------|------|
| `stack.sh` が途中で失敗する | `/opt/stack/logs/stack.sh.log` を確認する |
| メモリ不足 | 最低8GB RAMのフレーバーを使用する |
| Horizonにアクセスできない | セキュリティグループで80ポートを確認する |
| DevStack再実行 | `su - stack -c "cd /opt/stack/devstack && ./unstack.sh && ./stack.sh"` |
