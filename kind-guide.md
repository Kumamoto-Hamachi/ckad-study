# kind 完全ガイド

## kindとは

**kind** (Kubernetes in Docker) は、Dockerコンテナを使用してローカルにKubernetesクラスターを作成するツールです。

## なぜkindを使うのか？

### 従来の課題
- **minikube**: 仮想マシンが必要、リソース消費が大きい
- **クラウドクラスター**: 費用がかかる、インターネット接続が必要
- **実機クラスター**: 複数マシンが必要、セットアップが複雑

### kindの利点
- **軽量**: Dockerコンテナベースで高速起動
- **無料**: 完全にローカルで動作
- **マルチノード**: 複数ノードクラスターの構築が簡単
- **CI/CD**: 継続的インテグレーションに最適

## インストール

### 前提条件
- Docker Desktop または Docker Engine
- kubectl

### kindのインストール
省略

### インストール確認
```bash
kind version
# kind v0.20.0 go1.20.4 linux/amd64
```

## 基本的な使い方

### 1. シンプルなクラスター作成
```bash
# デフォルトクラスター作成
kind create cluster

# 名前付きクラスター作成
kind create cluster --name my-cluster

# 作成確認
kubectl cluster-info --context kind-my-cluster
```

### 2. クラスター一覧表示
```bash
kind get clusters
# kind
# my-cluster
```

### 3. クラスターの削除
```bash
# デフォルトクラスター削除
kind delete cluster

# 名前付きクラスター削除
kind delete cluster --name my-cluster
```

### 4. クラスター情報確認
```bash
# ノード確認
kubectl get nodes

# クラスター情報
kubectl cluster-info

# 全リソース確認
kubectl get all --all-namespaces
```

## 高度な設定

### マルチノードクラスター

#### 設定ファイル作成
```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```

#### マルチノードクラスター作成
```bash
kind create cluster --name multi-node --config kind-config.yaml

# ノード確認
kubectl get nodes
# NAME                       STATUS   ROLES           AGE   VERSION
# multi-node-control-plane   Ready    control-plane   2m    v1.27.3
# multi-node-worker          Ready    <none>          2m    v1.27.3
# multi-node-worker2         Ready    <none>          2m    v1.27.3
# multi-node-worker3         Ready    <none>          2m    v1.27.3
```

### カスタムKubernetesバージョン

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.27.3@sha256:3966ac761ae0136263ffdb6cfd4db23ef8a83cba8a463690e98317add2c9ba72
- role: worker
  image: kindest/node:v1.27.3@sha256:3966ac761ae0136263ffdb6cfd4db23ef8a83cba8a463690e98317add2c9ba72
```

### ポートマッピング（LoadBalancer対応）

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
```

```bash
kind create cluster --name lb-cluster --config kind-config.yaml
```

### Ingress設定

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
```

## 実践的なワークフロー

### 1. 開発環境セットアップ
```bash
# 開発用クラスター作成
kind create cluster --name dev

# kubectlコンテキスト切り替え
kubectl config use-context kind-dev

# 必要なリソースをデプロイ
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

### 2. CKAD試験対策
```bash
# 試験用クラスター作成
kind create cluster --name ckad-practice

# 練習問題の実行
kubectl run nginx --image=nginx
kubectl expose pod nginx --port=80
kubectl get pods,services
```

### 3. マルチクラスター管理
```bash
# 複数クラスター作成
kind create cluster --name cluster-a
kind create cluster --name cluster-b

# コンテキスト切り替え
kubectl config use-context kind-cluster-a
kubectl config use-context kind-cluster-b

# 全クラスター削除
kind delete clusters cluster-a cluster-b
```

## よく使うコマンド集

### クラスター管理
```bash
# クラスター作成
kind create cluster --name <cluster-name>

# 設定ファイルを使用
kind create cluster --name <cluster-name> --config <config-file>

# クラスター削除
kind delete cluster --name <cluster-name>

# 全クラスター削除
kind delete clusters $(kind get clusters)

# クラスター一覧
kind get clusters
```

### Docker統合
```bash
# kindが使用するDockerコンテナ確認
docker ps -a | grep kind

# kindコンテナのログ確認
docker logs kind-control-plane

# kindコンテナに直接アクセス
docker exec -it kind-control-plane bash
```

### イメージ管理
```bash
# ローカルイメージをkindにロード
kind load docker-image my-app:latest --name my-cluster

# tarファイルからイメージをロード
kind load image-archive my-app.tar --name my-cluster
```

## トラブルシューティング

### よくある問題と解決方法

#### 1. クラスター作成に失敗する
```bash
# Dockerが起動しているか確認
docker ps

# kindのログ確認
kind create cluster --name debug --verbosity=1

# 古いクラスターの削除
kind delete cluster --name kind
```

#### 2. kubectlがクラスターに接続できない
```bash
# コンテキスト確認
kubectl config get-contexts

# 正しいコンテキストに切り替え
kubectl config use-context kind-kind

# クラスター情報確認
kubectl cluster-info
```

#### 3. LoadBalancerが動作しない
```bash
# MetalLBをインストール
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml

# MetalLB設定
kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: example
  namespace: metallb-system
spec:
  addresses:
  - 172.19.255.200-172.19.255.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system
EOF
```

#### 4. ストレージが動作しない
```bash
# デフォルトStorageClass確認
kubectl get storageclass

# 標準StorageClassを設定
kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### デバッグコマンド
```bash
# kindクラスターのステータス確認
kind get nodes --name my-cluster

# Dockerコンテナのリソース使用量確認
docker stats $(docker ps --format "table {{.Names}}" | grep kind)

# kindコンテナ内のログ確認
docker exec kind-control-plane journalctl -u kubelet
```

## 設定ファイルの例

### 完全な設定例
```yaml
# production-like.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: production-like
networking:
  # デフォルトCNI無効
  disableDefaultCNI: false
  # Pod subnet
  podSubnet: "10.244.0.0/16"
  # Service subnet
  serviceSubnet: "10.96.0.0/16"
nodes:
- role: control-plane
  image: kindest/node:v1.27.3@sha256:3966ac761ae0136263ffdb6cfd4db23ef8a83cba8a463690e98317add2c9ba72
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
  - containerPort: 443
    hostPort: 443
  extraMounts:
  - hostPath: /tmp/kind-data
    containerPath: /data
- role: worker
  image: kindest/node:v1.27.3@sha256:3966ac761ae0136263ffdb6cfd4db23ef8a83cba8a463690e98317add2c9ba72
- role: worker
  image: kindest/node:v1.27.3@sha256:3966ac761ae0136263ffdb6cfd4db23ef8a83cba8a463690e98317add2c9ba72
```

## 便利なTips

### 1. エイリアス設定
```bash
# ~/.bashrc または ~/.zshrc
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kc='kubectl config use-context'
alias kcc='kubectl config current-context'
```

### 2. 自動補完設定

```bash
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'complete -F __start_kubectl k' >>~/.bashrc
```

### 3. 便利なスクリプト
```bash
#!/bin/bash
# quick-cluster.sh
CLUSTER_NAME=${1:-dev}
echo "Creating cluster: $CLUSTER_NAME"
kind create cluster --name $CLUSTER_NAME
kubectl config use-context kind-$CLUSTER_NAME
echo "Cluster $CLUSTER_NAME is ready!"
```
