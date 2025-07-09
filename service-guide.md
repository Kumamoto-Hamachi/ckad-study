# Service 完全ガイド

## Serviceとは

ServiceはKubernetesにおいて、**Pod群への安定したアクセスを提供する仕組み**です。

## なぜServiceが必要なのか？

### 問題：Pod直接アクセスの制約

#### 1. 不安定なIPアドレス
```bash
# Podは再作成されるたびにIPアドレスが変わる
kubectl get pods -o wide
# NAME                    READY   STATUS    RESTARTS   AGE   IP
# nginx-deployment-abc123 1/1     Running   0          1m    10.244.0.10
# nginx-deployment-def456 1/1     Running   0          1m    10.244.0.11

# Pod再作成後
kubectl get pods -o wide
# NAME                    READY   STATUS    RESTARTS   AGE   IP
# nginx-deployment-xyz789 1/1     Running   0          30s   10.244.0.15  # IPが変わった！
```

#### 2. 複数Podへのロードバランシング問題
```bash
# 3つのPodがあっても、どれにアクセスすべきか？
kubectl get pods -o wide
# NAME                    READY   STATUS    RESTARTS   AGE   IP
# nginx-deployment-abc123 1/1     Running   0          1m    10.244.0.10
# nginx-deployment-def456 1/1     Running   0          1m    10.244.0.11
# nginx-deployment-xyz789 1/1     Running   0          1m    10.244.0.12
```

#### 3. 手動でのPodアクセス（port-forward）
```bash
# 特定のPodを指定してポートフォワード
kubectl port-forward nginx-deployment-abc123 8080:80
# このコマンドは1つのPodにのみ接続
# Podが削除されると接続が切れる
# 負荷分散されない
```

### 解決：Serviceの恩恵

#### 1. 安定したアクセスポイント
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

```bash
# Serviceは固定のIPアドレスを持つ
kubectl get svc
# NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# nginx-service  ClusterIP   10.96.199.100   <none>        80/TCP    5m
```

#### 2. 自動ロードバランシング
```bash
# Serviceが自動的に複数のPodに負荷分散
curl http://nginx-service.default.svc.cluster.local
# リクエストが3つのPodに順次振り分けられる
```

#### 3. サービスディスカバリー
```bash
# DNS名でアクセス可能
curl http://nginx-service                    # 同じnamespace内
curl http://nginx-service.default           # namespace指定
curl http://nginx-service.default.svc.cluster.local  # FQDN
```

## Serviceなしでのアクセス方法（port-forward）

### 基本的な使い方

```bash
# 1. 特定のPodにport-forward
kubectl port-forward nginx-deployment-abc123 8080:80
# ローカルの8080ポートをPodの80ポートにフォワード

# 2. ブラウザまたはcurlでアクセス
curl http://localhost:8080
```

### より実践的なport-forward

```bash
# 1. 動的なPod選択
kubectl port-forward deployment/nginx-deployment 8080:80
# Deploymentの任意のPodにフォワード

# 2. バックグラウンド実行
kubectl port-forward deployment/nginx-deployment 8080:80 &
# バックグラウンドで実行

# 3. 特定のインターフェースにバインド
kubectl port-forward --address 0.0.0.0 deployment/nginx-deployment 8080:80
# 全てのインターフェースからアクセス可能
```

### port-forwardの制約

| 制約 | 説明 | 影響 |
|------|------|------|
| **単一Pod接続** | 1つのPodにのみ接続 | 負荷分散されない |
| **一時的接続** | コマンド終了で切断 | 本番環境に不適切 |
| **手動操作** | 毎回コマンド実行が必要 | 自動化困難 |
| **Pod障害時** | 接続Podが死ぬと切断 | 可用性が低い |
| **開発用途** | デバッグ・テスト用 | 本番運用不可 |

### port-forwardの使用例

```bash
# 開発中のデバッグ
kubectl port-forward pod/mysql-0 3306:3306
mysql -h localhost -P 3306 -u root -p

# 管理用ダッシュボードへのアクセス
kubectl port-forward deployment/grafana 3000:3000
# ブラウザでhttp://localhost:3000にアクセス

# 一時的なAPI確認
kubectl port-forward service/api-service 8080:80
curl http://localhost:8080/api/health
```

## Serviceの3つのタイプ

### 1. ClusterIP（デフォルト）
**用途**: クラスター内部通信

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
spec:
  type: ClusterIP  # 省略可能（デフォルト）
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

**特徴**:
- クラスター内部のみアクセス可能
- 固定IPアドレス（10.96.0.0/12の範囲）
- DNS名でアクセス可能

```bash
# 確認方法
kubectl get svc nginx-clusterip
# NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# nginx-clusterip  ClusterIP   10.96.199.100   <none>        80/TCP    1m

# クラスター内からのアクセス
kubectl run -it --rm debug --image=busybox --restart=Never -- wget -qO- http://nginx-clusterip
```

### 2. NodePort
**用途**: 外部からのアクセス（開発・テスト環境）

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080  # 30000-32767の範囲
```

**特徴**:
- 各ノードの指定ポートからアクセス可能
- ClusterIPの機能も含む
- ポート範囲: 30000-32767

```bash
# 確認方法
kubectl get svc nginx-nodeport
# NAME             TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# nginx-nodeport   NodePort   10.96.199.101   <none>        80:30080/TCP   1m

# 外部からのアクセス
curl http://<NODE-IP>:30080
```

### 3. LoadBalancer
**用途**: 本番環境での外部公開

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

**特徴**:
- クラウドプロバイダーのLBを自動作成
- 外部IPアドレスが割り当てられる
- NodePortとClusterIPの機能も含む

```bash
# 確認方法
kubectl get svc nginx-loadbalancer
# NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
# nginx-loadbalancer   LoadBalancer   10.96.199.102   203.0.113.100   80:31234/TCP   2m

# 外部からのアクセス
curl http://203.0.113.100
```

## 実践的な使用例

### 1. マイクロサービス間通信
```yaml
# フロントエンド → バックエンドAPI
apiVersion: v1
kind: Service
metadata:
  name: backend-api
spec:
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080
```

```bash
# フロントエンドからAPIを呼び出し
curl http://backend-api:8080/api/users
```

### 2. データベース接続
```yaml
# アプリケーション → データベース
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
```

```bash
# アプリケーションからDB接続
mysql -h mysql-service -P 3306 -u app -p
```

### 3. 外部公開（段階的アプローチ）
```bash
# 1. 開発環境：port-forward
kubectl port-forward deployment/webapp 8080:80

# 2. テスト環境：NodePort
kubectl expose deployment webapp --type=NodePort --port=80

# 3. 本番環境：LoadBalancer
kubectl expose deployment webapp --type=LoadBalancer --port=80
```

## トラブルシューティング

### Service接続確認
```bash
# 1. Service状態確認
kubectl get svc nginx-service
kubectl describe svc nginx-service

# 2. エンドポイント確認
kubectl get endpoints nginx-service
# NAME            ENDPOINTS                                   AGE
# nginx-service   10.244.0.10:80,10.244.0.11:80,10.244.0.12:80   5m

# 3. DNS解決確認
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup nginx-service

# 4. 接続テスト
kubectl run -it --rm debug --image=busybox --restart=Never -- wget -qO- http://nginx-service
```

### よくある問題と解決方法

#### 1. エンドポイントが空
```bash
kubectl get endpoints nginx-service
# NAME            ENDPOINTS   AGE
# nginx-service   <none>      5m
```
**原因**: selectorとPodのlabelが一致していない
**解決**: labelの確認と修正

#### 2. 外部からアクセスできない
```bash
# NodePortの確認
kubectl get svc nginx-nodeport
# ファイアウォール設定確認
```

#### 3. DNS解決できない
```bash
# CoreDNSの確認
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

## 重要なポイント

### Serviceの選択指針
- **ClusterIP**: 内部通信のみ
- **NodePort**: 開発・テスト環境の外部公開
- **LoadBalancer**: 本番環境の外部公開

### port-forwardの使い分け
- **一時的なデバッグ**: `kubectl port-forward`
- **継続的なアクセス**: Service作成

### ベストプラクティス
1. 本番環境では必ずServiceを使用
2. port-forwardは開発・デバッグのみ
3. 適切なServiceタイプを選択
4. DNS名を活用したサービス間通信

Serviceは**Kubernetesの根幹となる機能**であり、安定したアプリケーション運用には不可欠です。
