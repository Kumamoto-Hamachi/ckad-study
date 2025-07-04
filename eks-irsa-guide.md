# EKS IRSA Guide

## IRSA（IAM Roles for Service Accounts）とは
EKSでKubernetes ServiceAccountにAWS IAMロールを関連付ける仕組み。PodがAWS APIを直接呼び出せるようになる。

## 基本的な仕組み
1. **OIDCプロバイダー**: EKSクラスターがOIDCプロバイダーを提供
2. **IAMロール**: AWS側でTrust関係を設定したIAMロールを作成
3. **ServiceAccount**: KubernetesでIAMロールARNをアノテーションで指定
4. **Pod実行**: 自動的にAWS認証情報が注入される

## Terraformでの実装

### 1. EKSクラスターのOIDCプロバイダー
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

### 2. ServiceAccount用IAMロール
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

### 3. ServiceAccountの作成
```hcl
# Kubernetes Provider
provider "kubernetes" {
  host                   = aws_eks_cluster.main.endpoint
  cluster_ca_certificate = base64decode(aws_eks_cluster.main.certificate_authority[0].data)
  token                  = data.aws_eks_cluster_auth.main.token
}

data "aws_eks_cluster_auth" "main" {
  name = aws_eks_cluster.main.name
}

# ServiceAccount
resource "kubernetes_service_account" "my_sa" {
  metadata {
    name      = "my-service-account"
    namespace = "default"
    annotations = {
      "eks.amazonaws.com/role-arn" = aws_iam_role.pod_role.arn
    }
  }
}
```

### 4. Podでの利用
```hcl
# Pod定義
resource "kubernetes_pod" "my_pod" {
  metadata {
    name = "my-pod"
  }
  
  spec {
    service_account_name = kubernetes_service_account.my_sa.metadata[0].name
    
    container {
      name  = "app"
      image = "amazon/aws-cli"
      command = ["sleep", "3600"]
      
      env {
        name  = "AWS_REGION"
        value = "us-west-2"
      }
    }
  }
}
```

## 実用的な例

### 1. S3アクセス用ServiceAccount
```hcl
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

# ServiceAccount
resource "kubernetes_service_account" "s3_sa" {
  metadata {
    name      = "s3-service-account"
    namespace = "default"
    annotations = {
      "eks.amazonaws.com/role-arn" = aws_iam_role.s3_access_role.arn
    }
  }
}
```

### 2. Secrets Manager用ServiceAccount
```hcl
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

# ServiceAccount
resource "kubernetes_service_account" "secrets_sa" {
  metadata {
    name      = "secrets-service-account"
    namespace = "default"
    annotations = {
      "eks.amazonaws.com/role-arn" = aws_iam_role.secrets_role.arn
    }
  }
}
```

## 検証とデバッグ

### 1. Pod内でのAWS認証確認
```bash
# Pod内でAWS認証情報を確認
kubectl exec -it my-pod -- aws sts get-caller-identity

# 環境変数の確認
kubectl exec -it my-pod -- env | grep AWS
```

### 2. IAMロールの動作確認
```bash
# S3アクセステスト
kubectl exec -it s3-pod -- aws s3 ls s3://my-app-bucket

# Secrets Managerアクセステスト
kubectl exec -it secrets-pod -- aws secretsmanager get-secret-value --secret-id my-app-secret
```

### 3. Terraformでの出力設定
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
- 必要最小限のAWS権限のみ付与
- リソースレベルでの権限制限

### 2. Namespace分離
- 異なるアプリケーションは異なるNamespaceで実行
- NamespaceごとにServiceAccountを分離

### 3. 監査とログ
- CloudTrailでAPI呼び出しを監査
- EKSコントロールプレーンのログを有効化

### 4. セキュリティ
- Trust関係の条件を厳密に設定
- 定期的な権限の見直し

## トラブルシューティング

### よくある問題
1. **権限エラー**: IAMポリシーが不足
2. **認証エラー**: Trust関係の設定ミス
3. **ServiceAccountエラー**: アノテーションの設定ミス

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
IRSAを使用することで、EKS上のPodが安全にAWS APIを利用できます。Terraformを使用することで、インフラとKubernetesリソースを一元管理できます。
