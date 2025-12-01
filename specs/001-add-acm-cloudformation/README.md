# 001-add-acm-cloudformation

nagiyu 共通 ACM 証明書を CloudFormation で定義する機能の仕様ディレクトリ。

## 機能概要

nagiyu の各システム共通で利用する AWS ACM（AWS Certificate Manager）証明書を CloudFormation テンプレートで定義し、GitHub Actions から自動デプロイできるようにする。

### 主な機能

- **ACM 証明書の CloudFormation 定義**: ルートドメインとワイルドカードサブドメイン（例: `example.com`, `*.example.com`）を単一の証明書で管理
- **DNS 検証用レコード情報の出力**: CloudFormation の `Outputs` として検証用レコード（RecordName, RecordType, RecordValue 等）を機械可読な形式で出力
- **GitHub Actions によるデプロイ自動化**: Secrets（`ROOT_DOMAIN`）を利用したワークフロー実行
- **運用手順の提供**: 外部 DNS への検証レコード追加手順を含む運用ドキュメント

### MVP（Minimum Viable Product）

1. **Phase 1 (Setup)** と **Phase 2 (Foundational)** を完了し、`cloudformation/acm.yaml` の最小実装で `Outputs` を取得可能にする
2. **Phase 3 (User Story 1)** を実装して GitHub Actions からデプロイし、運用者が手動で DNS 検証を行って ACM 証明書が `ISSUED` になることを確認する

これが MVP の達成条件であり、User Story 2/3 は MVP 達成後に順次追加する。

## 目的

- 外部 DNS から AWS リソース（CloudFront 等）へドメインを向けられるようにする
- ルートドメインとサブドメインを1つのオリジンで管理できるようにする
- デプロイを GitHub Actions で自動化し、運用負荷を軽減する
- 証明書は CloudFront 等のグローバル配信との互換性のため、常に `us-east-1` で発行する

## 主要ファイル一覧

| ファイル | 役割 |
|----------|------|
| `spec.md` | 機能仕様書。ユーザーストーリー、機能要件、成功基準を定義 |
| `plan.md` | 実装計画書。技術コンテキスト、プロジェクト構成、Constitution チェック結果 |
| `tasks.md` | タスク一覧。フェーズ別の実装タスクと依存関係を定義 |
| `research.md` | 設計判断の記録。リージョン、DNS 検証、Outputs 仕様等の決定事項 |
| `data-model.md` | データモデル定義。ACM Certificate、CloudFormation Stack、DNS Record 等のエンティティ |
| `quickstart.md` | クイックスタートガイド。デプロイから証明書発行までの簡易手順 |
| `contracts/outputs.schema.json` | CloudFormation Outputs の JSON Schema。必須フィールドと型を定義 |

## 関連リソース（実装対象）

タスク完了後に作成・更新されるファイル:

| パス | 役割 | タスク |
|------|------|--------|
| `cloudformation/acm.yaml` | ACM 証明書の CloudFormation テンプレート | T001, T004, T005, T012 |
| `.github/workflows/deploy-acm.yml` | GitHub Actions デプロイワークフロー | T002, T008, T014 |
| `scripts/validate_acm_outputs.sh` | Outputs 検証スクリプト | T010 |
| `docs/cloudfront/using-acm.md` | CloudFront での証明書利用手順 | T013 |

## 次の実装候補

MVP 達成後に追加を検討する機能:

- **User Story 2**: ルートとサブドメインを単一オリジンで管理（CloudFront 設定との連携）
- **User Story 3**: 自動デプロイと運用（スタック更新時のログ出力、IAM ポリシー JSON 例）
- 外部 DNS プロバイダ向けの自動化プラグイン（将来的な拡張）

## 参照

- [spec.md](./spec.md) - 詳細な機能仕様
- [tasks.md](./tasks.md) - タスク一覧と進捗管理
- [quickstart.md](./quickstart.md) - クイックスタートガイド
