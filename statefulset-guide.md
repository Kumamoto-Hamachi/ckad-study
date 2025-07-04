# StatefulSet 簡易ガイド

## StatefulSetとは

StatefulSetは、ステートフルなアプリケーション（データベース、キューシステムなど）を管理するためのKubernetesリソースです。

## DeploymentとStatefulSetの違い

| 項目 | Deployment | StatefulSet |
|------|------------|-------------|
| Pod名 | ランダム | 順序付き (mysql-0, mysql-1, mysql-2) |
| 起動順序 | 並列 | 順次 (0→1→2) |
| ストレージ | 共有可能 | 各Pod専用 |
| ネットワークID | 不安定 | 安定 |
| 用途 | ステートレス | ステートフル |

## StatefulSetの主な特徴

### 1. 順序付きデプロイメント
- Pod名は `<StatefulSet名>-<序数>` の形式
- 起動は序数順（0番から順次）
- 削除は逆順（最大序数から順次）

### 2. 安定したネットワークID
- 各PodはHeadless Service経由で安定したDNS名を持つ
- `<Pod名>.<Service名>.<namespace>.svc.cluster.local`

### 3. 永続ストレージ
- `volumeClaimTemplates`で各Podに専用のPVCを作成
- Pod再作成時も同じPVCを使用

## 基本的な使い方

### 1. StatefulSetの作成
```bash
kubectl apply -f statefulset.yaml
```

### 2. Pod確認
```bash
kubectl get pods -l app=mysql
# mysql-0, mysql-1, mysql-2 の順序で表示
```

### 3. スケーリング
```bash
kubectl scale statefulset mysql-statefulset --replicas=5
# mysql-3, mysql-4が順次作成される
```

### 4. 削除
```bash
kubectl delete statefulset mysql-statefulset
# mysql-2, mysql-1, mysql-0の順で削除
```

## 重要なポイント

### Headless Service
- `clusterIP: None`で定義
- StatefulSetのPodに安定したネットワークIDを提供
- 必須の設定

### volumeClaimTemplates
- 各Podに専用のPVCを自動作成
- Pod削除時もPVCは残る（手動削除が必要）

### 更新戦略
- デフォルトは`RollingUpdate`
- `OnDelete`も選択可能

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
