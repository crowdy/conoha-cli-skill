# Kubernetesクラスター構築

## 概要

ConoHa VPS上にk3sを使ったKubernetesクラスターを構築するレシピ。マスターノード1台 + ワーカーノード2台の3ノード構成。

## 基本構成

- **ノード数**: 3（マスター1 + ワーカー2）
- **OS**: Ubuntu
- **k8sディストリビューション**: k3s
- **推奨フレーバー**: 2vCPU / 2GB RAM 以上

```
                    ┌─────────────┐
                    │ k8s-master-1│
                    │   (k3s      │
                    │   server)   │
                    └──────┬──────┘
                           │ K3S_TOKEN
              ┌────────────┼────────────┐
              │                         │
       ┌──────┴──────┐          ┌──────┴──────┐
       │k8s-worker-1 │          │k8s-worker-2 │
       │  (k3s agent)│          │  (k3s agent) │
       └─────────────┘          └──────────────┘
```

## 手順

### 1. 事前準備

セキュリティグループを作成し、必要なポートを開放する：

```bash
# セキュリティグループ作成
conoha network sg create --name k8s-sg

# SSH
conoha network sgr create --security-group-id <SG-ID> --direction ingress --protocol tcp --port-min 22 --port-max 22 --remote-ip 0.0.0.0/0

# Kubernetes API Server
conoha network sgr create --security-group-id <SG-ID> --direction ingress --protocol tcp --port-min 6443 --port-max 6443 --remote-ip 0.0.0.0/0

# kubelet
conoha network sgr create --security-group-id <SG-ID> --direction ingress --protocol tcp --port-min 10250 --port-max 10250 --remote-ip 0.0.0.0/0

# NodePort range
conoha network sgr create --security-group-id <SG-ID> --direction ingress --protocol tcp --port-min 30000 --port-max 32767 --remote-ip 0.0.0.0/0
```

### 2. サーバー作成

3台のサーバーを作成する：

```bash
conoha server create --name k8s-master-1 --flavor <フレーバーID> --image <UbuntuイメージID> --key-name my-key --security-group k8s-sg --wait
conoha server create --name k8s-worker-1 --flavor <フレーバーID> --image <UbuntuイメージID> --key-name my-key --security-group k8s-sg --wait
conoha server create --name k8s-worker-2 --flavor <フレーバーID> --image <UbuntuイメージID> --key-name my-key --security-group k8s-sg --wait
```

サーバーのIPを確認する：

```bash
conoha server show k8s-master-1 -o json
conoha server show k8s-worker-1 -o json
conoha server show k8s-worker-2 -o json
```

### 3. マスターノードセットアップ

k3sサーバーをインストールするスクリプト `k8s-master-setup.sh` を作成する：

```bash
#!/bin/bash
set -euo pipefail

# k3sサーバーインストール
curl -sfL https://get.k3s.io | sh -s - server \
  --write-kubeconfig-mode 644 \
  --tls-san "$MASTER_IP"

# インストール完了待機
until kubectl get nodes; do
  sleep 2
done

echo "k3s server installed successfully"
```

実行する：

```bash
MASTER_IP=$(conoha server show k8s-master-1 -o json | jq -r '.addresses | to_entries[0].value[] | select(.version == 4) | .addr')
conoha server deploy k8s-master-1 --script k8s-master-setup.sh --env MASTER_IP=$MASTER_IP
```

### 4. ジョイントークン取得

マスターからワーカー用のトークンを取得する：

```bash
K3S_TOKEN=$(ssh root@$MASTER_IP "cat /var/lib/rancher/k3s/server/node-token")
```

### 5. ワーカーノードセットアップ

k3sエージェントをインストールするスクリプト `k8s-worker-setup.sh` を作成する：

```bash
#!/bin/bash
set -euo pipefail

# k3sエージェントインストール
curl -sfL https://get.k3s.io | K3S_URL="https://${MASTER_IP}:6443" K3S_TOKEN="$JOIN_TOKEN" sh -s - agent

echo "k3s agent installed successfully"
```

各ワーカーで実行する：

```bash
conoha server deploy k8s-worker-1 --script k8s-worker-setup.sh --env MASTER_IP=$MASTER_IP --env JOIN_TOKEN=$K3S_TOKEN
conoha server deploy k8s-worker-2 --script k8s-worker-setup.sh --env MASTER_IP=$MASTER_IP --env JOIN_TOKEN=$K3S_TOKEN
```

### 6. 動作確認

```bash
ssh root@$MASTER_IP "kubectl get nodes"
```

期待される出力：

```
NAME           STATUS   ROLES                  AGE   VERSION
k8s-master-1   Ready    control-plane,master   5m    v1.xx.x+k3s1
k8s-worker-1   Ready    <none>                 2m    v1.xx.x+k3s1
k8s-worker-2   Ready    <none>                 2m    v1.xx.x+k3s1
```

kubeconfigをローカルにコピーする：

```bash
scp root@$MASTER_IP:/etc/rancher/k3s/k3s.yaml ~/.kube/config
sed -i "s/127.0.0.1/$MASTER_IP/" ~/.kube/config
kubectl get nodes
```

## カスタマイズ

### ワーカーノード数の変更

ワーカーを追加する場合、手順2で追加のサーバーを作成し、手順5を繰り返す。

### kubeadmを使用する場合

k3sの代わりにkubeadmを使用する場合、マスターセットアップスクリプトを以下に変更する：

```bash
#!/bin/bash
set -euo pipefail

# コンテナランタイムとkubeadmインストール
apt-get update
apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' > /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl containerd
apt-mark hold kubelet kubeadm kubectl

# kubeadm init
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=$MASTER_IP

# kubeconfig設定
mkdir -p /root/.kube
cp /etc/kubernetes/admin.conf /root/.kube/config

# Flannelインストール
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

ジョイントークンの取得コマンドも変更する：

```bash
JOIN_CMD=$(ssh root@$MASTER_IP "kubeadm token create --print-join-command")
```

## トラブルシューティング

| 問題 | 対処 |
|------|------|
| ワーカーがNotReady | `kubectl describe node <名前>` でイベントを確認する |
| k3sインストールが失敗する | サーバーのメモリが2GB以上あるか確認する（`conoha flavor show <ID>`） |
| API Serverに接続できない | セキュリティグループで6443ポートが開放されているか確認する |
| ノード間通信ができない | 全ノードが同じセキュリティグループに属しているか確認する |
