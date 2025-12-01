# Feature Specification: nagiyu 共通 ACM を CloudFormation で定義

**Feature Branch**: `001-add-acm-cloudformation`  
**Created**: 2025-12-01  
**Status**: Draft  
**Input**: ユーザー説明: "nagiyu の各システム共通で利用する AWS の ACM を CloudFormation で定義する。外部 DNS から AWS リソースへドメインを向けられるようにしたい。オリジンは、ルートとサブドメインの両方を1つで管理したい。デプロイは GitHub Actions で自動化する。ルートのドメインは GitHub Actions の Secrets で指定する。"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - ドメインを外部DNSからAWSへ向ける（Priority: P1）

運用チームは、既存の外部DNSサービスで管理しているドメイン（ルートドメインおよびサブドメイン）を、nagiyu の共通インフラに向けられるようにしたい。

**Why this priority**: ドメインの向け替えができなければ HTTPS 証明書や CDN 配信が利用できないため、最優先で対応する必要がある。

**Independent Test**: GitHub Actions を実行して CloudFormation スタックを作成し、指定された外部DNSで案内された DNS レコードを追加すると、ACM 証明書が発行（ISSUED）され、HTTPS 経由でアクセスできることを確認できる。

**Acceptance Scenarios**:

1. **Given** GitHub Actions に `ROOT_DOMAIN` シークレットが登録されている, **When** CI をトリガーして CloudFormation をデプロイする, **Then** 共通 ACM が作成され、検証用の DNS レコードが出力される。
2. **Given** 外部DNS に検証用 DNS レコードを追加した後, **When** DNS が伝播したら, **Then** ACM 証明書状態が `ISSUED` となり、指定されたドメインで TLS が有効になる。

---

### User Story 2 - ルートとサブドメインを単一オリジンで管理（Priority: P2）

運用チームは、ルートドメインと複数のサブドメインを1つのオリジン（例: 共通の CloudFront またはロードバランサ）で管理できるようにしたい。

**Why this priority**: 管理コストを下げ、証明書と配信設定を一元化するため。

**Independent Test**: ドメイン（例: example.com）とサブドメイン（例: www.example.com, api.example.com）を同一の配信設定に追加し、各ホスト名で期待するコンテンツが返ることを確認できる。

**Acceptance Scenarios**:

1. **Given** 共通オリジン設定がデプロイされている, **When** ルートとサブドメインを代替名として追加する, **Then** いずれのホスト名でも正しい証明書が提供される。

---

### User Story 3 - 自動デプロイと運用（Priority: P3）

運用者は GitHub Actions による自動デプロイで CloudFormation を更新し、証明書の更新（例: 期限切れ対応）やスタック変更を自動化したい。

**Why this priority**: 手動作業を減らし、再現可能なデプロイを実現するため。

**Independent Test**: GitHub Actions のジョブを実行し、CloudFormation の変更が成功することを確認する。

**Acceptance Scenarios**:

1. **Given** GitHub Actions がトリガーされる, **When** 変更をプッシュする, **Then** CloudFormation が適用されて変更が反映される。

---

### Edge Cases

- 外部DNS が ALIAS/ANAME をサポートしない場合、ルートドメインの向け方が制約される（代替策を要検討）。
- DNS 検証が手動でしか行えない外部DNSプロバイダの場合、完全自動化が困難になる。

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: CloudFormation テンプレートで共通の ACM 証明書リソースを定義できること。
- **FR-002**: GitHub Actions から指定されたドメイン名（Secrets 経由）を使ってデプロイできること。
- **FR-003**: DNS 検証用のレコード（CNAME または TXT）を CloudFormation の出力として取得できること。
    - CloudFormation の `Outputs` は機械で解釈しやすい構造とし、少なくとも以下を出力することを必須とする:
        - `RecordName` - 検証用レコードの名前（例: `_abcde.example.com.`）
        - `RecordType` - レコード種別（`CNAME` 或いは `TXT`）
        - `RecordValue` - 検証用レコードの値（CNAME のターゲットまたは TXT の文字列）
        - `DomainName` - 検証対象のドメイン（例: `example.com`）
        - `CertificateArn` - 作成された ACM 証明書の ARN
    - 参考（任意）: `TTL`（秒）と `Description`（人間向け説明）を追加出力すると運用上の可読性が上がる。
- **FR-004**: ルートドメインとサブドメインを同一の証明書／オリジンで管理できること（Subject Alternative Names または代替名で扱う）。
- **FR-005**: デプロイ結果（スタック作成/更新の成功・失敗）を GitHub Actions のログに残せること。
- **FR-006**: ACM 証明書は CloudFront 等のグローバル配信での互換性確保のため、常に `us-east-1` にて発行すること（固定）。テンプレートで発行リージョンを上書きできない設計とする。
- **FR-007**: 外部 DNS プロバイダ側での検証レコード自動追加は想定しない。CloudFormation は検証用のレコード情報を出力し、運用者が外部DNSに手動で追加する運用手順を提供すること。
- **FR-008**: サブドメインの運用負荷を下げるため、単一の ACM 証明書にルート（例: `example.com`）とワイルドカード（例: `*.example.com`）を代替名（SAN）として含めてリクエスト・管理できること。CloudFormation テンプレートは両方のドメイン名を同一証明書リソースとして指定できることを想定する。
- **FR-009**: テンプレートを GitHub Actions からデプロイする際に必要な最小 IAM 権限を定義し、運用手順に記載すること（例: CloudFormation によるリソース作成、ACM 操作、必要な `iam:PassRole` の範囲指定など）。

### IAM: 最小権限（example）

運用上の推奨（実装前に実際のリソース ARN に限定すること）として、デプロイ用の IAM ポリシーに含めるべき最小アクションの例を示す:

- CloudFormation
    - `cloudformation:CreateStack`
    - `cloudformation:UpdateStack`
    - `cloudformation:DescribeStacks`
    - `cloudformation:DescribeStackEvents`
    - `cloudformation:DeleteStack` (運用で必要な場合のみ)

- ACM
    - `acm:RequestCertificate`
    - `acm:DescribeCertificate`
    - `acm:ListCertificates`
    - `acm:DeleteCertificate` (運用で必要な場合のみ)

- IAM
    - `iam:PassRole` - CloudFormation が利用するロールに対してのみ許可（ワイルドカードは避け、対象ロールの ARN を限定すること）

- S3 (テンプレートを S3 から取得する場合のみ)
    - `s3:GetObject`

- STS / 監査用
    - `sts:GetCallerIdentity` (実行主体確認)

運用上の注意:
- リソース指定（ARN）で最小化すること。例えば `iam:PassRole` は CloudFormation 用に明示した Role ARN のみ許可する。
- 必要に応じて CloudFormation のスタック操作を行う特定のユーザー/ロールにポリシーをアタッチし、GitHub Actions 側の資格情報はそのユーザーに限定する。

（注）上記は一般例であり、最終的なポリシーはテンプレート中で参照するリソースや組織ポリシーに合わせて最小化してください。

### Key Entities *(include if feature involves data)*

- **ACM Certificate**: ドメイン名（例: root とサブドメイン）と検証ステータスを持つリソース。
- **CloudFormation Stack**: 証明書、出力（検証レコード）、必要な IAM ロール等を包含するテンプレートの実体。
- **DNS Record**: 検証用 TXT/CNAME レコードや、外部DNSで指す A/ALIAS/CNAME レコード。
- **GitHub Actions Secret**: ルートドメイン名を含むシークレット（例: `ROOT_DOMAIN`）。

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 90% のテスト対象ドメインで、GitHub Actions ジョブ開始（ワークフロー実行の `started_at` タイムスタンプ）を起点、ACM 証明書のステータスが `ISSUED` になった時点を終点として、60 分以内に `ISSUED` になることを目標とする。DNS レコードの手動追加時刻や DNS 伝播時間は別メトリクスで追跡し、本指標の対象外とする。測定方法: 各ワークフロー実行で `started_at` を記録し、定期的に `DescribeCertificate` を実行して `ISSUED` になったタイムスタンプを取得、差分を算出して成功割合を評価する。
- **SC-002**: デプロイ後、ルートおよび主要サブドメインへの HTTPS アクセスが 3 分以内に確立する（DNS 伝播を除くローカル確認）。
- **SC-003**: GitHub Actions の実行で CloudFormation スタックが成功（非エラー）する割合が 95% 以上であること。
- **SC-004**: 運用者が手順に従って10分以内に外部DNS側の検証レコードを追加できること（自動化不可の場合の運用目標）。

## Assumptions

- ルートドメイン名は GitHub Actions のシークレット（デフォルト名: `ROOT_DOMAIN`）で提供される。
- デフォルトの証明書検証方式は DNS 検証とする。
- グローバル配信サービスを利用する場合、証明書の発行リージョンが要件となるため、本仕様では `us-east-1` に固定して発行する（テンプレートで上書き不可）。

- デプロイ認証方式は Secrets のみを使用する（OIDC は採用しない）。

## Clarifications

### Session 2025-12-01

- Q: デプロイ方法（OIDC と Secrets の扱い） → A: `Secrets のみを使用する`（OIDC は採用しない）。
- Q: ワイルドカードとルートの同時発行方式 → A: `単一の ACM 証明書にルートとワイルドカードを含める`（単一証明書での SAN 管理を採用）。
- Q: 成功基準 `SC-001` の起点/終点定義 → A: `起点 = GitHub Actions ジョブ開始, 終点 = ACM が ISSUED`（Option A を採用）。
- Q: CloudFormation Outputs の具体フォーマット → A: `RecordName, RecordType, RecordValue, DomainName, CertificateArn を必須出力`（Option A を採用）。
- Q: IAM の最小権限 → A: `cloudformation:CreateStack/Update/Describe, acm:RequestCertificate/Describe, iam:PassRole (限定ARN), sts:GetCallerIdentity`（最小アクション群を採用）。
- Q: 証明書発行リージョンの扱い（`us-east-1` の可変性） → A: 常に `us-east-1` にて発行する（固定、テンプレートで上書き不可）。

## Deliverables

- `specs/001-add-acm-cloudformation/spec.md`（この仕様書）
- CloudFormation テンプレート（ACM 証明書リソース、必要であれば検証用出力）
- GitHub Actions ワークフロー定義（シークレット `ROOT_DOMAIN` を参照してスタックをデプロイ）
- 実運用向け手順書（外部DNSに対する検証レコード追加手順）

## Next Steps

1. 上記の `NEEDS CLARIFICATION` に対して回答を得る（優先順位: リージョン、DNS 自動化の可否）。
2. テンプレートを作成し、最小限の検証用ドメインで実際にデプロイして証明書発行を確認する。
3. 自動化が難しい DNS プロバイダ向けに手順書を整備する。
