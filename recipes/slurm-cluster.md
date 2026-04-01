# Slurmクラスター構築

## 概要

ConoHa VPS上にSlurmジョブスケジューラを使ったHPCクラスターを構築するレシピ。コントローラーノード1台 + コンピュートノード2台の構成。

## 基本構成

- **ノード数**: 3（コントローラー1 + コンピュート2）
- **OS**: Ubuntu
- **ジョブスケジューラ**: Slurm
- **共有ストレージ**: NFS（コントローラーがエクスポート）

```
                    ┌──────────────────┐
                    │ slurm-controller │
                    │  ┌────────────┐  │
                    │  │  slurmctld │  │
                    │  │  slurmdbd  │  │
                    │  │  NFS server│  │
                    │  └────────────┘  │
                    │  /shared (NFS)   │
                    └────────┬─────────┘
                             │
                ┌────────────┼────────────┐
                │                         │
       ┌────────┴────────┐       ┌────────┴────────┐
       │ slurm-compute-1 │       │ slurm-compute-2 │
       │  ┌────────────┐ │       │  ┌────────────┐ │
       │  │   slurmd   │ │       │  │   slurmd   │ │
       │  │ NFS client │ │       │  │ NFS client │ │
       │  └────────────┘ │       │  └────────────┘ │
       │  /shared (mount)│       │  /shared (mount)│
       └─────────────────┘       └─────────────────┘
```

## 手順

### 1. 事前準備

セキュリティグループを作成する：

```bash
conoha network sg create --name slurm-sg

# SSH
conoha network sgr create --security-group-id <SG-ID> --direction ingress --protocol tcp --port-min 22 --port-max 22 --remote-ip 0.0.0.0/0

# Slurm通信ポート
conoha network sgr create --security-group-id <SG-ID> --direction ingress --protocol tcp --port-min 6817 --port-max 6819 --remote-ip 0.0.0.0/0

# NFS
conoha network sgr create --security-group-id <SG-ID> --direction ingress --protocol tcp --port-min 2049 --port-max 2049 --remote-ip 0.0.0.0/0

# RPC (NFS関連)
conoha network sgr create --security-group-id <SG-ID> --direction ingress --protocol tcp --port-min 111 --port-max 111 --remote-ip 0.0.0.0/0
```

### 2. サーバー作成

```bash
conoha server create --name slurm-controller --flavor <フレーバーID> --image <UbuntuイメージID> --key-name my-key --security-group slurm-sg --wait
conoha server create --name slurm-compute-1 --flavor <フレーバーID> --image <UbuntuイメージID> --key-name my-key --security-group slurm-sg --wait
conoha server create --name slurm-compute-2 --flavor <フレーバーID> --image <UbuntuイメージID> --key-name my-key --security-group slurm-sg --wait
```

各サーバーのIPを確認する：

```bash
CTRL_IP=$(conoha server show slurm-controller -o json | jq -r '.addresses | to_entries[0].value[] | select(.version == 4) | .addr')
COMP1_IP=$(conoha server show slurm-compute-1 -o json | jq -r '.addresses | to_entries[0].value[] | select(.version == 4) | .addr')
COMP2_IP=$(conoha server show slurm-compute-2 -o json | jq -r '.addresses | to_entries[0].value[] | select(.version == 4) | .addr')
```

### 3. 全ノード共通セットアップ

共通セットアップスクリプト `slurm-common.sh` を作成する：

```bash
#!/bin/bash
set -euo pipefail

# /etc/hostsにエントリ追加
cat >> /etc/hosts <<EOF
$CTRL_IP slurm-controller
$COMP1_IP slurm-compute-1
$COMP2_IP slurm-compute-2
EOF

# Slurmインストール
apt-get update
apt-get install -y slurm-wlm slurm-client munge

# Mungeキー配置（コントローラーから取得、または新規生成）
if [ "$NODE_ROLE" = "controller" ]; then
  create-munge-key
else
  echo "$MUNGE_KEY_BASE64" | base64 -d > /etc/munge/munge.key
fi
chmod 400 /etc/munge/munge.key
chown munge:munge /etc/munge/munge.key
systemctl enable munge
systemctl restart munge

echo "Common setup complete for $NODE_ROLE"
```

全ノードで実行する：

```bash
conoha server deploy slurm-controller --script slurm-common.sh --env CTRL_IP=$CTRL_IP --env COMP1_IP=$COMP1_IP --env COMP2_IP=$COMP2_IP --env NODE_ROLE=controller
```

コントローラーからMungeキーを取得する：

```bash
MUNGE_KEY=$(ssh root@$CTRL_IP "base64 /etc/munge/munge.key")
```

コンピュートノードで共通セットアップを実行する：

```bash
conoha server deploy slurm-compute-1 --script slurm-common.sh --env CTRL_IP=$CTRL_IP --env COMP1_IP=$COMP1_IP --env COMP2_IP=$COMP2_IP --env NODE_ROLE=compute --env MUNGE_KEY_BASE64="$MUNGE_KEY"
conoha server deploy slurm-compute-2 --script slurm-common.sh --env CTRL_IP=$CTRL_IP --env COMP1_IP=$COMP1_IP --env COMP2_IP=$COMP2_IP --env NODE_ROLE=compute --env MUNGE_KEY_BASE64="$MUNGE_KEY"
```

### 4. コントローラーセットアップ

コントローラーセットアップスクリプト `slurm-controller-setup.sh` を作成する：

```bash
#!/bin/bash
set -euo pipefail

# NFSサーバーセットアップ
apt-get install -y nfs-kernel-server
mkdir -p /shared
chmod 777 /shared
echo "/shared *(rw,sync,no_subtree_check,no_root_squash)" > /etc/exports
exportfs -ra
systemctl enable nfs-kernel-server
systemctl restart nfs-kernel-server

# Slurm設定ファイル作成
cat > /etc/slurm/slurm.conf <<EOF
ClusterName=conoha-cluster
SlurmctldHost=slurm-controller

MpiDefault=none
ProctrackType=proctrack/linuxproc
ReturnToService=1
SlurmctldPidFile=/run/slurmctld.pid
SlurmdPidFile=/run/slurmd.pid
SlurmdSpoolDir=/var/spool/slurmd
SlurmUser=slurm
StateSaveLocation=/var/spool/slurmctld

SchedulerType=sched/backfill
SelectType=select/cons_tres

# ノード定義
NodeName=slurm-compute-1 NodeAddr=$COMP1_IP CPUs=$CPUS RealMemory=$MEMORY State=UNKNOWN
NodeName=slurm-compute-2 NodeAddr=$COMP2_IP CPUs=$CPUS RealMemory=$MEMORY State=UNKNOWN

# パーティション定義
PartitionName=batch Nodes=slurm-compute-[1-2] Default=YES MaxTime=INFINITE State=UP
EOF

# slurmctld起動
mkdir -p /var/spool/slurmctld
chown slurm:slurm /var/spool/slurmctld
systemctl enable slurmctld
systemctl restart slurmctld

echo "Controller setup complete"
```

実行する：

```bash
conoha server deploy slurm-controller --script slurm-controller-setup.sh --env COMP1_IP=$COMP1_IP --env COMP2_IP=$COMP2_IP --env CPUS=2 --env MEMORY=2000
```

### 5. コンピュートノードセットアップ

コンピュートセットアップスクリプト `slurm-compute-setup.sh` を作成する：

```bash
#!/bin/bash
set -euo pipefail

# NFSマウント
apt-get install -y nfs-common
mkdir -p /shared
mount $CTRL_IP:/shared /shared
echo "$CTRL_IP:/shared /shared nfs defaults 0 0" >> /etc/fstab

# コントローラーからslurm.confをコピー
scp -o StrictHostKeyChecking=no root@$CTRL_IP:/etc/slurm/slurm.conf /etc/slurm/slurm.conf

# slurmd起動
mkdir -p /var/spool/slurmd
chown slurm:slurm /var/spool/slurmd
systemctl enable slurmd
systemctl restart slurmd

echo "Compute node setup complete"
```

各コンピュートノードで実行する：

```bash
conoha server deploy slurm-compute-1 --script slurm-compute-setup.sh --env CTRL_IP=$CTRL_IP
conoha server deploy slurm-compute-2 --script slurm-compute-setup.sh --env CTRL_IP=$CTRL_IP
```

### 6. 動作確認

```bash
# ノード状態確認
ssh root@$CTRL_IP "sinfo"
```

期待される出力：

```
PARTITION AVAIL TIMELIMIT NODES STATE NODELIST
batch*    up    infinite  2     idle  slurm-compute-[1-2]
```

テストジョブを実行する：

```bash
ssh root@$CTRL_IP "srun --nodes=2 hostname"
```

期待される出力：

```
slurm-compute-1
slurm-compute-2
```

バッチジョブのテスト：

```bash
ssh root@$CTRL_IP "cat > /shared/test.sh << 'SCRIPT'
#!/bin/bash
echo \"Hello from \$(hostname) at \$(date)\" > /shared/output_\$(hostname).txt
SCRIPT
chmod +x /shared/test.sh
sbatch --nodes=2 --ntasks=2 /shared/test.sh"
```

## カスタマイズ

### コンピュートノードの追加

1. 新しいサーバーを作成する
2. 共通セットアップ → コンピュートセットアップを実行する
3. コントローラーの `slurm.conf` にノード定義を追加する
4. `scontrol reconfigure` を実行する

### GPUノードの使用

GPUフレーバーを選択し、`slurm.conf` のノード定義に `Gres=gpu:1` を追加する。

### ジョブスケジューラの調整

`slurm.conf` の `SchedulerType` と `SelectType` を変更する：
- `sched/backfill`: デフォルト、バックフィルスケジューリング
- `select/cons_tres`: CPU/メモリ/GPUの個別割り当て

## トラブルシューティング

| 問題 | 対処 |
|------|------|
| ノードがdown状態 | `scontrol update nodename=<名前> state=idle` で復旧を試みる |
| Munge認証エラー | 全ノードで同じMungeキーが配置されているか確認する |
| NFSマウントが失敗する | セキュリティグループでNFSポート(2049, 111)を確認する |
| ジョブがpending | `squeue` と `sinfo` でノードの状態を確認する |
