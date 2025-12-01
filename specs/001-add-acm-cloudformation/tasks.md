---
description: "Tasks for 001-add-acm-cloudformation"
---

# タスク: nagiyu 共通 ACM を CloudFormation で定義

**入力**: `specs/001-add-acm-cloudformation/` 以下の設計資料

## フェーズ 1: Setup (共有インフラ初期化)

- [ ] T001 [P] `cloudformation/acm.yaml` のディレクトリとプレースホルダテンプレートを作成する（パス: `cloudformation/acm.yaml`）
- [ ] T002 `GitHub Actions` ワークフローの雛形を作成する（パス: `.github/workflows/deploy-acm.yml`）
- [ ] T003 [P] `specs/001-add-acm-cloudformation/README.md` に機能概要と主要ファイル一覧を追加する（パス: `specs/001-add-acm-cloudformation/README.md`）

---

## フェーズ 2: Foundational (ブロッキング要件)

- [ ] T004 `cloudformation/acm.yaml` に ACM 証明書リソースを実装する（パス: `cloudformation/acm.yaml`）
- [ ] T005 [P] CloudFormation の `Outputs` を仕様に沿って実装する（必須出力: `RecordName`, `RecordType`, `RecordValue`, `DomainName`, `CertificateArn`）（パス: `cloudformation/acm.yaml`）
- [ ] T006 `specs/001-add-acm-cloudformation/iam/deploy-policy.md` にデプロイに必要な最小 IAM 権限を記載する（パス: `specs/001-add-acm-cloudformation/iam/deploy-policy.md`）
- [ ] T007 [P] `specs/001-add-acm-cloudformation/contracts/outputs.schema.json` を検証対象として用いるドキュメントを整備する（パス: `specs/001-add-acm-cloudformation/contracts/outputs.schema.json`）

---

## フェーズ 3: User Story 1 - ドメインを外部DNSからAWSへ向ける（Priority: P1）

**目標**: GitHub Actions で CloudFormation をデプロイし、出力された検証用 DNS レコード情報を用いて ACM 証明書が `ISSUED` になるまでの手順を提供する。

**独立テスト基準**: `.github/workflows/deploy-acm.yml` を実行し、CloudFormation の `Outputs` を確認、外部DNS に指定されたレコードを追加すると ACM が `ISSUED` になる。

- [ ] T008 [US1] `.github/workflows/deploy-acm.yml` を実装する（Secrets: `ROOT_DOMAIN` を参照して `aws cloudformation deploy` を実行）（パス: `.github/workflows/deploy-acm.yml`）
- [ ] T009 [US1] `specs/001-add-acm-cloudformation/quickstart.md` を更新して実行手順と `aws cloudformation deploy` の具体例を記載する（パス: `specs/001-add-acm-cloudformation/quickstart.md`）
- [ ] T010 [US1] `scripts/validate_acm_outputs.sh` を追加し、CloudFormation Outputs の `CertificateArn` を引数に `aws acm describe-certificate` で状態確認できるようにする（パス: `scripts/validate_acm_outputs.sh`）
- [ ] T011 [US1] 運用手順書 `specs/001-add-acm-cloudformation/operation.md` を作成し、出力の読み方と外部DNSへのレコード追加手順を明記する（パス: `specs/001-add-acm-cloudformation/operation.md`）

---

## フェーズ 4: User Story 2 - ルートとサブドメインを単一オリジンで管理（Priority: P2）

**目標**: 単一の ACM 証明書でルートとワイルドカードサブドメインを管理できるようにし、オリジン（例: CloudFront）で利用できるようにする。

**独立テスト基準**: 同一の証明書 ARN を CloudFront 設定に指定し、ルート・サブドメイン両方で TLS が有効になる。

- [ ] T012 [P] [US2] `cloudformation/acm.yaml` の証明書リクエストで `SubjectAlternativeNames` にルートドメインとワイルドカード（例: `example.com`, `*.example.com`）を指定できるようにする（パス: `cloudformation/acm.yaml`）
- [ ] T013 [US2] ドキュメント `docs/cloudfront/using-acm.md` を作成し、CloudFront での証明書利用手順を記載する（パス: `docs/cloudfront/using-acm.md`）

---

## フェーズ 5: User Story 3 - 自動デプロイと運用（Priority: P3）

**目標**: GitHub Actions による CloudFormation の更新自動化（スタック更新・ログ記録）と、運用時の再現手順を整備する。

**独立テスト基準**: ワークフローを実行して CloudFormation の更新が成功し、ログにスタック操作結果が記録される。

- [ ] T014 [US3] `.github/workflows/deploy-acm.yml` にスタック更新時の成功/失敗判定と出力のログ出力を追加する（パス: `.github/workflows/deploy-acm.yml`）
- [ ] T015 [P] [US3] `specs/001-add-acm-cloudformation/iam/deploy-policy.json` を作成し、実際に使用する最小 IAM ポリシーの JSON 例を追加する（パス: `specs/001-add-acm-cloudformation/iam/deploy-policy.json`）

---

## 最終フェーズ: Polish & 横断的懸案

- [ ] T016 [P] `specs/001-add-acm-cloudformation/tasks.md` のチェック（本ファイル）とフォーマット検証を行う（パス: `specs/001-add-acm-cloudformation/tasks.md`）
- [ ] T017 [P] `specs/001-add-acm-cloudformation/contracts/outputs.schema.json` に対する Outputs 検証スクリプトを追加する（パス: `scripts/validate_outputs_schema.sh`）
- [ ] T018 ドキュメント整備: `specs/001-add-acm-cloudformation/README.md` に MVP の定義と次の実装候補を記載する（パス: `specs/001-add-acm-cloudformation/README.md`）

---

## 依存関係（簡易）

- Setup (T001-T003) → Foundational (T004-T007) を完了後、各 User Story (T008-T015) を実装可能
- User Story 間は独立性を保つ（US1 を MVP として先に完了推奨）

## 実装戦略（MVP 優先）

1. Phase1 と Phase2 をまず完了し、`cloudformation/acm.yaml` の最小実装で `Outputs` を取得可能にする。
2. Phase3 (US1) を実装して GitHub Actions からデプロイし、運用者が手動で DNS 検証を行って `ISSUED` になることを確認する（ここが MVP）。
3. US2/US3 を順次追加し、自動化と運用ドキュメントを整備する。
