# research.md

## Decision: 証明書発行リージョン
- Decision: ACM 証明書は常に `us-east-1` で発行する。
- Rationale: CloudFront 等のグローバル配信と互換性を確保するため、かつ本仕様の明示的要件であるため固定とする。
- Alternatives considered: テンプレートでリージョンを可変にする案は検討したが、誤用によるグローバル配信問題を避けるため却下。

## Decision: DNS 検証の自動化
- Decision: 外部DNSプロバイダ側での自動レコード追加は想定しない（運用者が手動で追加）。
- Rationale: 外部DNSプロバイダは各社APIや認証方式が多様で、一般的な実装では信頼できない。手動での追加を前提にし、CloudFormation 側は検証用レコードを出力して運用手順を提示する。
- Alternatives considered: 一部プロバイダ向けに自動化スクリプトを追加する案（Route53以外）を検討。将来的な拡張としてプロバイダ別プラグインを追加可能。

## Decision: CloudFormation Outputs 仕様
- Decision: Outputs は機械可読なスキーマ（`RecordName`, `RecordType`, `RecordValue`, `DomainName`, `CertificateArn`）を必須とし、`TTL` と `Description` を任意で含める。
- Rationale: GitHub Actions や運用ツールが Outputs を解析して検証レコード案内を表示できるようにするため。JSON Schema を `specs/.../contracts/outputs.schema.json` として提供する。

## Decision: GitHub Actions デプロイ方式
- Decision: デプロイは GitHub Actions（Secrets 認証）で行う。デフォルトシークレット名は `ROOT_DOMAIN`。
- Rationale: 要求で OIDC を採用しない旨が既に決定されているため。Actions 内で `aws` CLI を使用して CloudFormation をデプロイするワークフローを作成する。

## Decision: 最小 IAM 権限
- Decision: ドキュメント中で例示された最小権限セットを採用する（CloudFormation スタック操作、ACM 証明書操作、限定的な `iam:PassRole`, `sts:GetCallerIdentity`）。
- Rationale: 最小権限の原則に従い、必要なアクションのみを許可する。実際のポリシーは運用環境の ARN に限定することを運用手順に明記する。
