# Node Scheduling Guide

## 概要
Podをどのノードで実行するかを制御する3つの主要な仕組み。

## 1. Taint & Toleration

### 特徴
- **除外ベース**: 特定のPodを拒否する仕組み
- **ノード側が主導**: ノードがTaintを設定してPodを拒否
- **デフォルト拒否**: TaintがあるノードはPodを受け入れない

### 仕組み
1. **Taint**: ノードに設定する「汚れ」
2. **Toleration**: Podに設定する「耐性」
3. **マッチング**: TolerationがTaintを許容する場合のみPodがスケジュール可能

### 使用例
```bash
# ノードにTaintを設定
kubectl taint nodes node1 key=value:NoSchedule

# Taintを削除
kubectl taint nodes node1 key=value:NoSchedule-
```

```yaml
# Pod定義でTolerationを設定
apiVersion: v1
kind: Pod
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

### Effect種類
- **NoSchedule**: 新しいPodをスケジュールしない
- **PreferNoSchedule**: 可能な限りスケジュールしない
- **NoExecute**: 実行中のPodも強制退去

## 2. NodeSelector

### 特徴
- **選択ベース**: 特定のノードを選択する仕組み
- **シンプル**: ラベルの完全一致のみ
- **厳密**: 条件に合わないノードは絶対に選ばない

### 仕組み
1. **ノードラベル**: ノードにラベルを設定
2. **NodeSelector**: PodでノードラベルをKey-Value形式で指定
3. **完全一致**: 指定したラベルが完全に一致するノードのみ選択

### 使用例
```bash
# ノードにラベルを設定
kubectl label nodes node1 disktype=ssd

# ラベルを削除
kubectl label nodes node1 disktype-
```

```yaml
# Pod定義でNodeSelectorを設定
apiVersion: v1
kind: Pod
spec:
  nodeSelector:
    disktype: ssd
```

## 3. NodeAffinity

### 特徴
- **選択ベース**: 特定のノードを選択する仕組み
- **柔軟**: 複雑な条件指定が可能
- **優先度**: 必須条件と推奨条件を設定可能

### 仕組み
1. **ノードラベル**: ノードにラベルを設定
2. **NodeAffinity**: Podで複雑な条件を指定
3. **柔軟マッチング**: In、NotIn、Exists、DoesNotExist等の演算子

### 使用例
```yaml
# Pod定義でNodeAffinityを設定
apiVersion: v1
kind: Pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
            - nvme
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - us-west-1a
```

### タイプ
- **requiredDuringSchedulingIgnoredDuringExecution**: 必須条件
- **preferredDuringSchedulingIgnoredDuringExecution**: 推奨条件

## 比較表

| 項目 | Taint/Toleration | NodeSelector | NodeAffinity |
|------|------------------|--------------|--------------|
| **アプローチ** | 除外ベース | 選択ベース | 選択ベース |
| **主導権** | ノード側 | Pod側 | Pod側 |
| **柔軟性** | 中 | 低 | 高 |
| **条件** | キー/値/Effect | キー=値 | 複雑な条件 |
| **デフォルト** | 拒否 | 制限なし | 制限なし |
| **用途** | 専用ノード/問題ノード | 簡単な選択 | 複雑な選択 |

## 使い分けのポイント

### Taint/Tolerationを使う場面
- **専用ノード**: 特定のアプリケーション専用
- **問題ノード**: メンテナンス中やハードウェア問題
- **リソース制限**: GPU専用ノードなど
- **除外したい**: 特定のPodを拒否したい

### NodeSelectorを使う場面
- **シンプルな選択**: 単純な条件で十分
- **既存システム**: 既にラベルが設定済み
- **厳密な要件**: 条件に合わないノードは絶対NG
- **学習目的**: 理解しやすい

### NodeAffinityを使う場面
- **複雑な条件**: 複数の条件を組み合わせ
- **優先度設定**: 必須条件と推奨条件
- **柔軟性が必要**: In、NotIn等の演算子
- **将来性**: NodeSelectorは非推奨予定

## 実践例

### 1. GPU専用ノード
```bash
# GPU搭載ノードにTaintを設定
kubectl taint nodes gpu-node1 gpu=true:NoSchedule

# GPU使用PodにTolerationを設定
tolerations:
- key: "gpu"
  value: "true"
  effect: "NoSchedule"
```

### 2. SSD搭載ノード選択
```bash
# SSD搭載ノードにラベル設定
kubectl label nodes ssd-node1 disktype=ssd

# データベースPodでSSDノードを選択
nodeSelector:
  disktype: ssd
```

### 3. 複雑な配置制御
```yaml
# 高性能ノード必須、特定ゾーン推奨
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
    - matchExpressions:
      - key: performance
        operator: In
        values: ["high", "premium"]
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    preference:
      matchExpressions:
      - key: zone
        operator: In
        values: ["us-west-1a"]
```

## トラブルシューティング

### よくある問題
1. **Podがスケジュールされない**
   - Taintに対応するTolerationがない
   - NodeSelectorの条件に合うノードがない
   - NodeAffinityの必須条件を満たすノードがない

2. **期待と異なるノードに配置**
   - NodeAffinityの推奨条件のweightが低い
   - 他の制約（リソース不足など）が優先

### デバッグコマンド
```bash
# ノードのTaint確認
kubectl describe nodes | grep -i taint

# ノードのラベル確認
kubectl get nodes --show-labels

# Podのスケジュール状況確認
kubectl describe pod <pod-name>
kubectl get events --field-selector involvedObject.name=<pod-name>
```

## まとめ
- **Taint/Toleration**: ノードが拒否、Podが許容を求める
- **NodeSelector**: Podがシンプルな条件でノード選択
- **NodeAffinity**: Podが複雑な条件でノード選択

適切な仕組みを選択して、効率的なPod配置を実現しましょう。
