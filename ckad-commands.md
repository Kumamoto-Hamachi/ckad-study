# CKAD Command Reference

## Pod Management

### Podの作成
- `kubectl run nginx --image=nginx`
- `kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml`

### Podの削除
- `kubectl delete pod nginx`
- `kubectl delete pod nginx --force --grace-period=0`

### Podの確認
- `kubectl get pods`
- `kubectl get pods -o wide`
- `kubectl describe pod nginx`

### Podのログ確認
- `kubectl logs nginx`
- `kubectl logs nginx -f`
- `kubectl logs nginx -c container-name`

### Podへの接続
- `kubectl exec -it nginx -- /bin/bash`
- `kubectl exec -it nginx -- sh`

## Deployment Management

### Deploymentの作成
- `kubectl create deployment nginx --image=nginx`
- `kubectl create deployment nginx --image=nginx --replicas=3`
- `kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deployment.yaml`

### Deploymentのスケーリング
- `kubectl scale deployment nginx --replicas=5`
- `kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80`

### Deploymentの更新
- `kubectl set image deployment/nginx nginx=nginx:1.20`
- `kubectl rollout restart deployment/nginx`

### Rollout管理
- `kubectl rollout status deployment/nginx`
- `kubectl rollout history deployment/nginx`
- `kubectl rollout undo deployment/nginx`
- `kubectl rollout undo deployment/nginx --to-revision=2`

## Service Management

### Serviceの作成
- `kubectl expose deployment nginx --port=80 --type=ClusterIP`
- `kubectl expose deployment nginx --port=80 --type=NodePort`
- `kubectl expose deployment nginx --port=80 --type=LoadBalancer`
- `kubectl create service clusterip nginx --tcp=80:80`

### Serviceの確認
- `kubectl get services`
- `kubectl get svc`
- `kubectl describe service nginx`

## ConfigMap Management

### ConfigMapの作成
- `kubectl create configmap app-config --from-literal=key1=value1`
- `kubectl create configmap app-config --from-file=config.properties`
- `kubectl create configmap app-config --from-env-file=.env`

### ConfigMapの確認
- `kubectl get configmaps`
- `kubectl describe configmap app-config`
- `kubectl get configmap app-config -o yaml`

## Secret Management

### Secretの作成
- `kubectl create secret generic db-secret --from-literal=username=admin`
- `kubectl create secret generic db-secret --from-file=username.txt`
- `kubectl create secret docker-registry regcred --docker-server=server --docker-username=user --docker-password=pass`

### Secretの確認
- `kubectl get secrets`
- `kubectl describe secret db-secret`
- `kubectl get secret db-secret -o yaml`

## Namespace Management

### Namespaceの作成
- `kubectl create namespace dev`
- `kubectl create ns dev`

### Namespaceの切り替え
- `kubectl config set-context --current --namespace=dev`
- `kubectl config use-context context-name`

### Namespace指定でのリソース操作
- `kubectl get pods -n dev`
- `kubectl get pods --all-namespaces`
- `kubectl get pods -A`

## Resource Management

### リソースの作成・更新
- `kubectl apply -f manifest.yaml`
- `kubectl create -f manifest.yaml`
- `kubectl replace -f manifest.yaml`
- `kubectl replace --force -f manifest.yaml`
- `kubectl apply -f .`

### リソースの削除
- `kubectl delete -f manifest.yaml`
- `kubectl delete pod,service nginx`
- `kubectl delete all --all`

### リソースの確認
- `kubectl get all`
- `kubectl get all -n namespace`
- `kubectl get events`
- `kubectl get events --sort-by=.metadata.creationTimestamp`

## Job Management

### Jobの作成
- `kubectl create job hello --image=busybox -- echo "Hello World"`
- `kubectl create job hello --image=busybox --dry-run=client -o yaml > job.yaml`

### CronJobの作成
- `kubectl create cronjob hello --image=busybox --schedule="*/1 * * * *" -- echo "Hello World"`

### Job確認
- `kubectl get jobs`
- `kubectl get cronjobs`
- `kubectl logs job/hello`

## Volume Management

### PersistentVolumeの確認
- `kubectl get pv`
- `kubectl get pvc`
- `kubectl describe pv pv-name`

## Network Management

### Ingressの確認
- `kubectl get ingress`
- `kubectl describe ingress ingress-name`

### NetworkPolicyの確認
- `kubectl get networkpolicy`
- `kubectl describe networkpolicy policy-name`

## Debugging

### リソースの詳細確認
- `kubectl describe pod nginx`
- `kubectl describe deployment nginx`
- `kubectl describe service nginx`

### ログの確認
- `kubectl logs nginx`
- `kubectl logs nginx --previous`
- `kubectl logs -l app=nginx`

### イベントの確認
- `kubectl get events`
- `kubectl get events --field-selector involvedObject.name=nginx`

### トラブルシューティング
- `kubectl get pods --show-labels`
- `kubectl get pods -o wide`
- `kubectl top pods`
- `kubectl top nodes`

## Utility Commands

### ラベルの操作
- `kubectl label pod nginx app=web`
- `kubectl label pod nginx app-`
- `kubectl get pods -l app=web`

### アノテーションの操作
- `kubectl annotate pod nginx description="web server"`
- `kubectl annotate pod nginx description-`

### リソースの編集
- `kubectl edit pod nginx`
- `kubectl edit deployment nginx`

### リソースの出力
- `kubectl get pod nginx -o yaml`
- `kubectl get pod nginx -o json`
- `kubectl get pods -o wide`

### コンテキストの管理
- `kubectl config get-contexts`
- `kubectl config current-context`
- `kubectl config use-context context-name`

## Quick Reference

### よく使うエイリアス
- `k` = `kubectl`
- `kgp` = `kubectl get pods`
- `kgs` = `kubectl get services`
- `kgd` = `kubectl get deployments`

### 時間短縮のためのオプション
- `--dry-run=client -o yaml` : YAMLファイル生成
- `--force --grace-period=0` : 強制削除
- `-o wide` : 詳細表示
- `-A` : 全Namespace
- `-l` : ラベルセレクタ
- `-f` : ログの継続表示
