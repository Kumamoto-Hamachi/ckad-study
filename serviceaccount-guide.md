# ServiceAccount Guide

## ServiceAccountとは
PodがKubernetes APIサーバーと通信するためのアイデンティティ（身元）を提供するリソース。人間のユーザーではなく、アプリケーションやサービスが使用する。
※人間が使うのはユーザーアカウント、マシンが使うのがServiceAccount

## 密接に関連するリソース

### Secret
- ServiceAccountの認証トークンを格納
- 自動的に作成されるか、手動で関連付け
- `type: kubernetes.io/service-account-token`

### Role/ClusterRole
- ServiceAccountに付与する権限を定義
- Role: Namespace内の権限
- ClusterRole: クラスター全体の権限

### RoleBinding/ClusterRoleBinding
- ServiceAccountとRole/ClusterRoleを紐付け
- 実際の権限付与を行う

## 基本的な使用パターン

### 1. ServiceAccountの作成
```bash
kubectl create serviceaccount my-sa
```

### 2. Roleの作成
```bash
kubectl create role pod-reader --verb=get,list --resource=pods
```

### 3. RoleBindingの作成
```bash
kubectl create rolebinding read-pods --role=pod-reader --serviceaccount=default:my-sa
```

### 4. PodでServiceAccountを使用
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: my-sa
  containers:
  - name: app
    image: nginx
```

## よく使用されるコマンド

### ServiceAccount管理
```bash
# 作成
kubectl create serviceaccount my-sa

# 確認
kubectl get serviceaccount
kubectl describe serviceaccount my-sa

# 削除
kubectl delete serviceaccount my-sa
```

### 権限管理
```bash
# Role作成
kubectl create role my-role --verb=get,list,watch --resource=pods

# ClusterRole作成
kubectl create clusterrole my-clusterrole --verb=get,list,watch --resource=nodes

# RoleBinding作成
kubectl create rolebinding my-binding --role=my-role --serviceaccount=default:my-sa

# ClusterRoleBinding作成
kubectl create clusterrolebinding my-cluster-binding --clusterrole=my-clusterrole --serviceaccount=default:my-sa
```

### 権限確認
```bash
# ServiceAccountの権限確認
kubectl auth can-i get pods --as=system:serviceaccount:default:my-sa

# 権限の一覧表示
kubectl auth can-i --list --as=system:serviceaccount:default:my-sa
```

## デフォルトのServiceAccount
- 各Namespaceに`default` ServiceAccountが自動作成
- Podで明示的に指定しない場合は`default`が使用される
- 最小限の権限のみ

## セキュリティのベストプラクティス
- 必要最小限の権限のみ付与
- デフォルトのServiceAccountは使用しない
- ServiceAccountごとに用途を明確にする
- 定期的な権限の見直し

## トラブルシューティング

### よくある問題
1. **権限不足エラー**
   - `kubectl auth can-i`で権限確認
   - RoleBindingの設定を確認

2. **ServiceAccountが見つからない**
   - Namespace内にServiceAccountが存在するか確認
   - Pod定義でのServiceAccount名のtypoを確認

3. **トークンの問題**
   - Secretが正しく関連付けられているか確認
   - トークンの有効期限を確認

### デバッグコマンド
```bash
# ServiceAccountの詳細確認
kubectl get sa my-sa -o yaml

# 関連するSecretの確認
kubectl get secrets

# RoleBindingの確認
kubectl get rolebinding
kubectl describe rolebinding my-binding

# Pod内でのServiceAccount確認
kubectl exec -it my-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

## 実践例

### 1. 読み取り専用のServiceAccount
```bash
# ServiceAccount作成
kubectl create sa readonly-sa

# 読み取り専用Role作成
kubectl create role readonly --verb=get,list,watch --resource=pods,services

# Binding作成
kubectl create rolebinding readonly-binding --role=readonly --serviceaccount=default:readonly-sa
```

### 2. 特定リソースへのアクセス
```bash
# ConfigMap専用ServiceAccount
kubectl create sa configmap-sa
kubectl create role configmap-role --verb=get,list,create,update --resource=configmaps
kubectl create rolebinding configmap-binding --role=configmap-role --serviceaccount=default:configmap-sa
```

## 注意点
- ServiceAccountの削除時は関連するRoleBinding/ClusterRoleBindingも確認
- Namespaceを跨ぐ権限にはClusterRoleBindingが必要
- ServiceAccount名は Pod定義で正確に指定する必要がある
