# StatefulSet 簡易ガイド

## StatefulSetとは

StatefulSetは、ステートフルなアプリケーション（データベース、キューシステムなど）を管理するためのKubernetesリソースです。

## なぜStatefulSetが必要なのか？

### 問題：Deploymentの制約
- **不安定なPod名**: `nginx-deployment-abc123-xyz789`（ランダム）
- **不安定なネットワーク**: Pod再作成時にIPアドレスが変わる
- **共有ストレージ**: 全Podが同じストレージを共有

### 解決：StatefulSetの特徴
- **安定したPod名**: `mysql-0`, `mysql-1`, `mysql-2`（固定）
- **安定したネットワーク**: 各Podに固有のDNS名
- **専用ストレージ**: 各Podに専用のストレージ

## DeploymentとStatefulSetの違い

| 項目 | Deployment | StatefulSet |
|------|------------|-------------|
| Pod名 | ランダム<br/>（nginx-abc123-xyz789） | 順序付き<br/>（mysql-0, mysql-1, mysql-2） |
| 起動順序 | 並列<br/>（全Pod同時起動） | 順次<br/>（0→1→2の順番） |
| ストレージ | 共有可能<br/>（複数Podが同じPVCを使用） | 各Pod専用<br/>（各Podに独自のPVC） |
| ネットワークID | 不安定<br/>（Pod再作成時に変わる） | 安定<br/>（固定のDNS名） |
| 用途 | ステートレス<br/>（Webサーバー、API） | ステートフル<br/>（データベース、キュー） |

## StatefulSetの主な特徴

### 1. 順序付きデプロイメント
- Pod名は `<StatefulSet名>-<序数>` の形式
- 起動は序数順（0番から順次）
- 削除は逆順（最大序数から順次）

### 2. 安定したネットワークID（Headless Service）

#### Headless Serviceとは
- **通常のService**: クラスターIP（例：10.96.0.1）を持ち、ロードバランシングを行う
- **Headless Service**: `clusterIP: None`で定義し、クラスターIPを持たない

#### 通常のServiceとの違い

```yaml
# 通常のService
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - port: 80
  # clusterIP: 10.96.0.1（自動割り当て）
```

```yaml
# Headless Service
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None  # ← これがポイント
  selector:
    app: mysql
  ports:
    - port: 3306
```

#### DNS解決の違い

| Service種類 | DNS解決結果 | 動作 |
|-------------|-------------|------|
| 通常のService | `nginx-service` → `10.96.0.1` | ServiceのIPを返す |
| Headless Service | `mysql-headless` → `10.244.0.1, 10.244.0.2, 10.244.0.3` | 各PodのIPを返す |

#### 各Podへの直接アクセス
- **DNS名**: `<Pod名>.<Service名>.<namespace>.svc.cluster.local`
- **例**:
  - `mysql-0.mysql-headless.default.svc.cluster.local`
  - `mysql-1.mysql-headless.default.svc.cluster.local`
  - `mysql-2.mysql-headless.default.svc.cluster.local`

### 3. 永続ストレージ
- `volumeClaimTemplates`で各Podに専用のPVCを作成
- Pod再作成時も同じPVCを使用

## 実践例：MySQL StatefulSetの構築

### 完全なYAMLファイル例

```yaml
# mysql-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - port: 3306
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-statefulset
spec:
  serviceName: mysql-headless
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

### 基本的な使い方

### 1. StatefulSetの作成
```bash
kubectl apply -f mysql-statefulset.yaml
```

### 2. Pod確認（順序付きで作成されることを確認）
```bash
kubectl get pods -l app=mysql -w
# 出力例：
# NAME                READY   STATUS    RESTARTS   AGE
# mysql-statefulset-0 1/1     Running   0          30s
# mysql-statefulset-1 1/1     Running   0          45s
# mysql-statefulset-2 1/1     Running   0          60s
```

### 3. DNS名の確認
```bash
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup mysql-statefulset-0.mysql-headless.default.svc.cluster.local
# 出力例：
# Name:      mysql-statefulset-0.mysql-headless.default.svc.cluster.local
# Address 1: 10.244.0.10 mysql-statefulset-0.mysql-headless.default.svc.cluster.local
```

### 4. 各Podへの直接アクセス
```bash
# mysql-0に直接接続
kubectl exec -it mysql-statefulset-0 -- mysql -u root -p

# mysql-1に直接接続
kubectl exec -it mysql-statefulset-1 -- mysql -u root -p
```

### 5. スケーリング
```bash
kubectl scale statefulset mysql-statefulset --replicas=5
# mysql-statefulset-3, mysql-statefulset-4が順次作成される
```

### 6. 削除
```bash
kubectl delete statefulset mysql-statefulset
# mysql-statefulset-4, mysql-statefulset-3, mysql-statefulset-2, mysql-statefulset-1, mysql-statefulset-0の順で削除
```

## 重要なポイント

### 1. Headless Service（必須）
```yaml
spec:
  clusterIP: None  # これを忘れると通常のServiceになる
  serviceName: mysql-headless  # StatefulSetでserviceNameを指定
```

### 2. volumeClaimTemplates（ストレージ管理）
```bash
# 各Podに自動作成されるPVC
kubectl get pvc
# 出力例：
# NAME                        STATUS   VOLUME   CAPACITY   ACCESS MODES   AGE
# mysql-data-mysql-statefulset-0   Bound    pv001    1Gi        RWO            5m
# mysql-data-mysql-statefulset-1   Bound    pv002    1Gi        RWO            5m
# mysql-data-mysql-statefulset-2   Bound    pv003    1Gi        RWO            5m
```

**注意**: Pod削除時もPVCは残る（データ永続化のため）
```bash
# PVCを手動で削除する場合
kubectl delete pvc mysql-data-mysql-statefulset-0
kubectl delete pvc mysql-data-mysql-statefulset-1
kubectl delete pvc mysql-data-mysql-statefulset-2
```

### 3. 更新戦略
- **デフォルト**: `RollingUpdate`（順次更新）
- **選択可能**: `OnDelete`（手動削除後に更新）

### 4. Pod間の通信例
```bash
# mysql-0からmysql-1に接続
kubectl exec -it mysql-statefulset-0 -- mysql -h mysql-statefulset-1.mysql-headless.default.svc.cluster.local -u root -p
```

## 実際の使用例

### データベース
- MySQL、PostgreSQL、MongoDB
- 各インスタンスに専用ストレージが必要

### 分散システム
- Kafka、Elasticsearch、Cassandra
- 各ノードに固有のIDが必要

### キューシステム
- RabbitMQ、Redis Cluster
- 順序付きデプロイメントが重要

## StatefulSetとPersistentVolumeの関係

### ストレージの階層構造
```
PersistentVolume (PV) ← 実際のストレージ
    ↑
PersistentVolumeClaim (PVC) ← ストレージの要求
    ↑
StatefulSet (volumeClaimTemplates) ← PVCの自動作成
```

### 違いと関係性

| リソース | 役割 | 管理者 |
|----------|------|--------|
| PersistentVolume | 実際のストレージ | クラスター管理者 |
| PersistentVolumeClaim | ストレージの要求 | 開発者 |
| StatefulSet | PVCの自動作成・管理 | 開発者 |

### StatefulSetの利点
- **自動PVC作成**: `volumeClaimTemplates`で各Podに専用PVCを自動作成
- **ライフサイクル管理**: Pod削除時もPVCは保持（データ永続化）
- **動的スケーリング**: レプリカ追加時に新しいPVCを自動作成

### 通常のDeploymentとの比較
```yaml
# Deployment: 手動でPVCを作成・管理
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-storage
spec:
  accessModes: ["ReadWriteMany"]
  resources:
    requests:
      storage: 1Gi

# StatefulSet: volumeClaimTemplatesで自動作成
spec:
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

## デバッグコマンド

```bash
# StatefulSet状態確認
kubectl describe statefulset mysql-statefulset

# Pod詳細確認
kubectl describe pod mysql-0

# PVC確認（自動作成されたPVCを確認）
kubectl get pvc
# mysql-data-mysql-0, mysql-data-mysql-1, mysql-data-mysql-2

# PV確認
kubectl get pv

# Service確認
kubectl get svc mysql-headless
```

# 参考
[StatefulSetっていつ使うの？PersistentVolumesでいいんじゃないの？ - blackawa](https://scrapbox.io/blackawa/StatefulSet%E3%81%A3%E3%81%A6%E3%81%84%E3%81%A4%E4%BD%BF%E3%81%86%E3%81%AE%EF%BC%9FPersistentVolumes%E3%81%A7%E3%81%84%E3%81%84%E3%82%93%E3%81%98%E3%82%83%E3%81%AA%E3%81%84%E3%81%AE%EF%BC%9F)
