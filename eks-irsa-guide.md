# EKS IRSA Guide

## IRSA（IAM Roles for Service Accounts）とは

EKS で Kubernetes ServiceAccount に AWS IAM ロールを関連付ける仕組み。Pod が AWS API を直接呼び出せるようになる。

## 基本的な仕組み

1. **OIDC プロバイダー**: EKS クラスターが OIDC プロバイダーを提供
2. **IAM ロール**: AWS 側で Trust 関係を設定した IAM ロールを作成
3. **ServiceAccount**: Kubernetes で IAM ロール ARN をアノテーションで指定
4. **Pod 実行**: 自動的に AWS 認証情報が注入される

## Terraform での実装

### 1. EKS クラスターの OIDC プロバイダー

```hcl
# EKSクラスター
resource "aws_eks_cluster" "main" {
  name     = "my-cluster"
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.29"

  vpc_config {
    subnet_ids = var.subnet_ids
  }
}

# OIDCプロバイダーのデータ取得
data "tls_certificate" "eks" {
  url = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

# OIDCプロバイダーの作成
resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.main.identity[0].oidc[0].issuer
}
```

### 2. ServiceAccount 用 IAM ロール

```hcl
# IAMロール
resource "aws_iam_role" "pod_role" {
  name = "my-pod-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRoleWithWebIdentity"
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.eks.arn
        }
        Condition = {
          StringEquals = {
            "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub" = "system:serviceaccount:default:my-service-account"
            "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })
}

# IAMポリシー
resource "aws_iam_policy" "pod_policy" {
  name        = "my-pod-policy"
  description = "Policy for my pod"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject"
        ]
        Resource = [
          "arn:aws:s3:::my-bucket/*"
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "ssm:GetParameter",
          "ssm:GetParameters"
        ]
        Resource = [
          "arn:aws:ssm:*:*:parameter/my-app/*"
        ]
      }
    ]
  })
}

# ロールにポリシーをアタッチ
resource "aws_iam_role_policy_attachment" "pod_policy" {
  role       = aws_iam_role.pod_role.name
  policy_arn = aws_iam_policy.pod_policy.arn
}
```

### 3. ServiceAccount の作成

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/my-pod-role
```

適用コマンド:

```bash
kubectl apply -f serviceaccount.yaml
```

### 4. Pod での利用

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: default
spec:
  serviceAccountName: my-service-account
  containers:
    - name: app
      image: amazon/aws-cli
      command: ["sleep", "3600"]
      env:
        - name: AWS_REGION
          value: us-west-2
```

適用コマンド:

```bash
kubectl apply -f pod.yaml
```

## 実用的な例

### 1. S3 アクセス用 ServiceAccount

````hcl
# S3専用のIAMロール
resource "aws_iam_role" "s3_access_role" {
  name = "s3-access-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRoleWithWebIdentity"
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.eks.arn
        }
        Condition = {
          StringEquals = {
            "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub" = "system:serviceaccount:default:s3-service-account"
            "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })
}

# S3アクセス用ポリシー
resource "aws_iam_policy" "s3_policy" {
  name = "s3-access-policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:ListBucket",
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = [
          aws_s3_bucket.my_bucket.arn,
          "${aws_s3_bucket.my_bucket.arn}/*"
        ]
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "s3_policy" {
  role       = aws_iam_role.s3_access_role.name
  policy_arn = aws_iam_policy.s3_policy.arn
}

# S3バケット
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-app-bucket"
}

# S3用ServiceAccount
```yaml
# s3-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-service-account
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/s3-access-role
````

適用コマンド:

```bash
kubectl apply -f s3-serviceaccount.yaml
```

### 2. Secrets Manager 用 ServiceAccount

````hcl
# Secrets Manager用IAMロール
resource "aws_iam_role" "secrets_role" {
  name = "secrets-manager-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRoleWithWebIdentity"
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.eks.arn
        }
        Condition = {
          StringEquals = {
            "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub" = "system:serviceaccount:default:secrets-service-account"
            "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })
}

# Secrets Manager用ポリシー
resource "aws_iam_policy" "secrets_policy" {
  name = "secrets-manager-policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ]
        Resource = [
          aws_secretsmanager_secret.app_secret.arn
        ]
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "secrets_policy" {
  role       = aws_iam_role.secrets_role.name
  policy_arn = aws_iam_policy.secrets_policy.arn
}

# Secrets Manager
resource "aws_secretsmanager_secret" "app_secret" {
  name = "my-app-secret"
}

resource "aws_secretsmanager_secret_version" "app_secret" {
  secret_id = aws_secretsmanager_secret.app_secret.id
  secret_string = jsonencode({
    db_password = "super-secret-password"
  })
}

# Secrets Manager用ServiceAccount
```yaml
# secrets-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secrets-service-account
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/secrets-manager-role
````

適用コマンド:

```bash
kubectl apply -f secrets-serviceaccount.yaml
```

## 検証とデバッグ

### 1. Pod 内での AWS 認証確認

```bash
# Pod内でAWS認証情報を確認
kubectl exec -it my-pod -- aws sts get-caller-identity

# 環境変数の確認
kubectl exec -it my-pod -- env | grep AWS
```

### 2. IAM ロールの動作確認

```bash
# S3アクセステスト
kubectl exec -it s3-pod -- aws s3 ls s3://my-app-bucket

# Secrets Managerアクセステスト
kubectl exec -it secrets-pod -- aws secretsmanager get-secret-value --secret-id my-app-secret
```

### 3. Terraform での出力設定

```hcl
# 出力値
output "cluster_oidc_issuer_url" {
  value = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

output "pod_role_arn" {
  value = aws_iam_role.pod_role.arn
}

output "service_account_name" {
  value = kubernetes_service_account.my_sa.metadata[0].name
}
```

## ベストプラクティス

### 1. 最小権限の原則

- 必要最小限の AWS 権限のみ付与
- リソースレベルでの権限制限

### 2. Namespace 分離

- 異なるアプリケーションは異なる Namespace で実行
- Namespace ごとに ServiceAccount を分離

### 3. 監査とログ

- CloudTrail で API 呼び出しを監査
- EKS コントロールプレーンのログを有効化

### 4. セキュリティ

- Trust 関係の条件を厳密に設定
- 定期的な権限の見直し

## トラブルシューティング

### よくある問題

1. **権限エラー**: IAM ポリシーが不足
2. **認証エラー**: Trust 関係の設定ミス
3. **ServiceAccount エラー**: アノテーションの設定ミス

### デバッグ手順

```bash
# ServiceAccountの確認
kubectl get sa -o yaml

# Podの環境変数確認
kubectl exec -it pod-name -- env | grep AWS

# CloudTrailでAPI呼び出し確認
# AWSコンソールまたはAWS CLIで確認
```

## まとめ

IRSA を使用することで、EKS 上の Pod が安全に AWS API を利用できます。Terraform を使用することで、インフラと Kubernetes リソースを一元管理できます。

# 参考

[Kubernetes の Role と RoleBinding でアクセス権を管理しよう｜ toshi](https://note.com/minato_kame/n/ne29b368fed24)
[IAM Roles for Service Accounts(IRSA)の仕組みを深堀りしてみた - APC 技術ブログ](https://techblog.ap-com.co.jp/entry/irsa-deep-dive)

> IRSA が最初にあったのですが、別の方法として EKS Pod Identity が昨年登場しました。
> 基本的には EKS Pod Identity を使用し、EKS Pod Identity が使用できない場面では IRSA を使うことが推奨されています

[EKS Pod Identity の仕組みを深堀りしてみた - APC 技術ブログ](https://techblog.ap-com.co.jp/entry/eks-pod-identity-deep-dive)
