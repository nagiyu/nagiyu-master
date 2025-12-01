# 最小 IAM 権限: ACM CloudFormation デプロイ

**パス**: `specs/001-add-acm-cloudformation/iam/deploy-policy.md`  
**参照**: `specs/001-add-acm-cloudformation/spec.md` (FR-009)

## 概要

GitHub Actions から `aws cloudformation deploy` を実行し、ACM 証明書をデプロイするために必要な最小限の IAM 権限を定義する。

## 必須権限

### CloudFormation

CloudFormation スタックの作成・更新・状態確認に必要な権限。

| アクション | 用途 | 必須 |
|------------|------|:----:|
| `cloudformation:CreateStack` | 新規スタックの作成 | ✓ |
| `cloudformation:UpdateStack` | 既存スタックの更新 | ✓ |
| `cloudformation:DescribeStacks` | スタック状態の確認 | ✓ |
| `cloudformation:DescribeStackEvents` | スタックイベントの取得（デバッグ用） | ✓ |
| `cloudformation:DeleteStack` | スタックの削除 | △ |
| `cloudformation:GetTemplate` | テンプレート取得（変更セット確認用） | △ |
| `cloudformation:CreateChangeSet` | 変更セットの作成 | △ |
| `cloudformation:DescribeChangeSet` | 変更セットの確認 | △ |
| `cloudformation:ExecuteChangeSet` | 変更セットの実行 | △ |
| `cloudformation:DeleteChangeSet` | 変更セットの削除 | △ |

**注記**:
- `DeleteStack` は運用でスタック削除が必要な場合のみ追加する
- 変更セット関連の権限は `--no-execute-changeset` オプションを使用する場合に必要

### ACM (AWS Certificate Manager)

ACM 証明書の発行・管理に必要な権限。

| アクション | 用途 | 必須 |
|------------|------|:----:|
| `acm:RequestCertificate` | 証明書のリクエスト | ✓ |
| `acm:DescribeCertificate` | 証明書の状態確認 | ✓ |
| `acm:ListCertificates` | 証明書一覧の取得 | ✓ |
| `acm:AddTagsToCertificate` | 証明書へのタグ付け | ✓ |
| `acm:DeleteCertificate` | 証明書の削除 | △ |
| `acm:ListTagsForCertificate` | 証明書タグの取得 | △ |

**注記**:
- `DeleteCertificate` は CloudFormation スタック削除時に自動で必要になる場合がある
- ACM 操作のリージョンは `us-east-1` に固定（CloudFront 互換性のため）

### IAM

CloudFormation が使用するロールへのパス権限。

| アクション | 用途 | 必須 |
|------------|------|:----:|
| `iam:PassRole` | CloudFormation 実行ロールへの権限委譲 | △ |

**注記**:
- `iam:PassRole` は CloudFormation がサービスロールを使用する場合のみ必要
- **重要**: `Resource` は対象ロールの ARN に限定すること（ワイルドカード `*` は避ける）
- 本テンプレートでは CloudFormation のサービスロールを使用しないため、通常は不要

### STS / 監査用

実行主体の確認に使用する権限。

| アクション | 用途 | 必須 |
|------------|------|:----:|
| `sts:GetCallerIdentity` | 実行ユーザー/ロールの確認 | △ |

**注記**:
- デバッグやログ記録のために推奨
- 必須ではないが、トラブルシューティング時に有用

### S3（オプション）

CloudFormation テンプレートを S3 から取得する場合に必要。

| アクション | 用途 | 必須 |
|------------|------|:----:|
| `s3:GetObject` | テンプレートファイルの取得 | △ |

**注記**:
- ローカルファイルまたはリポジトリ内のテンプレートを使用する場合は不要
- S3 バケットを使用する場合は、対象バケット/オブジェクトの ARN に限定すること

## 最小権限の推奨ポリシー

以下は、本機能のデプロイに必要な最小限の IAM ポリシー例である。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CloudFormationDeployment",
      "Effect": "Allow",
      "Action": [
        "cloudformation:CreateStack",
        "cloudformation:UpdateStack",
        "cloudformation:DescribeStacks",
        "cloudformation:DescribeStackEvents"
      ],
      "Resource": "arn:aws:cloudformation:us-east-1:ACCOUNT_ID:stack/PROJECT_NAME-acm-*"
    },
    {
      "Sid": "ACMCertificateManagement",
      "Effect": "Allow",
      "Action": [
        "acm:RequestCertificate",
        "acm:DescribeCertificate",
        "acm:ListCertificates",
        "acm:AddTagsToCertificate"
      ],
      "Resource": "*"
    }
  ]
}
```

**注記**:
- `ACCOUNT_ID` は実際の AWS アカウント ID に置き換えること
- スタック名のプレフィックス `PROJECT_NAME-acm-*` は実際のプロジェクト名（例: `nagiyu-acm-*`）に置き換えること
- ACM の `Resource` は `*` だが、これは ACM API の仕様上リソースレベルの制限が難しいため

## 運用時の追加権限

スタック削除や証明書削除が必要な運用フローでは、以下の権限を追加する。

```json
{
  "Sid": "StackAndCertificateDeletion",
  "Effect": "Allow",
  "Action": [
    "cloudformation:DeleteStack",
    "acm:DeleteCertificate"
  ],
  "Resource": "*"
}
```

**セキュリティ推奨事項**:
- 可能であれば `Resource` をワイルドカード `*` ではなく、特定のスタック ARN（例: `arn:aws:cloudformation:us-east-1:ACCOUNT_ID:stack/PROJECT_NAME-acm-*`）に限定する
- 証明書削除は慎重に行い、本番環境では削除権限を別ロールに分離することを検討する

## 最小権限の原則

1. **リソース限定**: 可能な限り `Resource` で対象リソースの ARN を指定する
2. **条件付き許可**: `Condition` を使用して、タグや環境による制限を検討する
3. **定期的な見直し**: 不要になった権限は速やかに削除する
4. **監査ログ**: CloudTrail でアクション履歴を記録し、不正アクセスを検知する

## GitHub Actions での設定

GitHub Actions で使用する場合、以下の Secrets を設定する:

- `AWS_ACCESS_KEY_ID`: デプロイ用 IAM ユーザーのアクセスキー ID
- `AWS_SECRET_ACCESS_KEY`: デプロイ用 IAM ユーザーのシークレットアクセスキー
- `ROOT_DOMAIN`: 証明書のルートドメイン名

**セキュリティ推奨事項**:
- デプロイ専用の IAM ユーザーを作成し、上記の最小権限のみを付与する
- アクセスキーは定期的にローテーションする
- 可能であれば、GitHub Actions の OIDC 認証への移行を検討する（本仕様では Secrets を使用）
    - OIDC の利点: 長期間有効なアクセスキーの保存が不要、トークンは自動的に期限切れ、より細かい条件制御が可能

## 参照

- [AWS CloudFormation アクション](https://docs.aws.amazon.com/service-authorization/latest/reference/list_awscloudformation.html)
- [AWS ACM アクション](https://docs.aws.amazon.com/service-authorization/latest/reference/list_awscertificatemanager.html)
- [最小権限の原則](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege)
