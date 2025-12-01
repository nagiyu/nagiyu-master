# Implementation Plan: [FEATURE]

**Branch**: `[###-feature-name]` | **Date**: [DATE] | **Spec**: [link]
**Input**: Feature specification from `/specs/[###-feature-name]/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

共通 ACM 証明書を CloudFormation テンプレートで定義し、GitHub Actions から `ROOT_DOMAIN` シークレットを用いてデプロイ可能にする。証明書は常に `us-east-1` で発行し、DNS 検証用のレコード情報は CloudFormation の `Outputs` として機械可読な形で出力する。運用手順として、GitHub Actions でスタックを作成後に出力された検証レコードを外部DNSに手動で追加し、検証完了で ACM が `ISSUED` となることを確認するワークフローを提供する。

## Technical Context

<!--
  ACTION REQUIRED: Replace the content in this section with the technical details
  for the project. The structure here is presented in advisory capacity to guide
  the iteration process.
-->

**Language/Version**: CloudFormation YAML (no runtime language required)
**Primary Dependencies**: AWS CloudFormation, AWS ACM (us-east-1), GitHub Actions
**Storage**: N/A
**Testing**: Integration tests via manual deploy; GitHub Actions job run validation
**Target Platform**: AWS (us-east-1 for ACM; deployment may run from any region via CloudFormation)
**Project Type**: Infrastructure as Code (CloudFormation template + CI workflow)
**Performance Goals**: 証明書発行（ACM が ISSUED）を60分以内（目標、DNS 伝播は除く）
**Constraints**: ACM 証明書は常に `us-east-1` で発行する（テンプレートで変更不可）。外部DNSの自動更新は想定しない（運用者が手動で検証レコードを追加）。
**Scale/Scope**: 単一 CloudFormation テンプレートでルートとワイルドカード（例: `example.com` と `*.example.com`）を含める想定

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

現状、リポジトリ内の `.specify/memory/constitution.md` はテンプレート（プレースホルダ）であり、具体的なガバナンスや必須ポリシーが定義されていません。したがって、Constitution に基づく自動判定は実行できず、ガバナンス検査結果は `NEEDS CLARIFICATION` として扱います。具体的な組織ポリシー（例: セキュリティ要件、承認フロー、テストゲート）を提示いただければ、再評価を行います。

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)
<!--
  ACTION REQUIRED: Replace the placeholder tree below with the concrete layout
  for this feature. Delete unused options and expand the chosen structure with
  real paths (e.g., apps/admin, packages/something). The delivered plan must
  not include Option labels.
-->

```text
# [REMOVE IF UNUSED] Option 1: Single project (DEFAULT)
src/
├── models/
├── services/
├── cli/
└── lib/

tests/
├── contract/
├── integration/
└── unit/

# [REMOVE IF UNUSED] Option 2: Web application (when "frontend" + "backend" detected)
backend/
├── src/
│   ├── models/
│   ├── services/
│   └── api/
└── tests/

frontend/
├── src/
│   ├── components/
│   ├── pages/
│   └── services/
└── tests/

# [REMOVE IF UNUSED] Option 3: Mobile + API (when "iOS/Android" detected)
api/
└── [same as backend above]

ios/ or android/
└── [platform-specific structure: feature modules, UI flows, platform tests]
```

**Structure Decision**: この作業はドキュメント中心のインフラ実装（CloudFormation テンプレート＋GitHub Actions）であるため、`specs/001-add-acm-cloudformation/` 以下に設計資料と契約（contracts）、テンプレート、ワークフロー定義を配置します。実際のテンプレート（例: `cloudformation/acm.yaml`）は別コミットで追加します。

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
