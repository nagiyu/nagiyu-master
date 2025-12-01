# CloudFormation Outputs 契約定義

**パス**: `specs/001-add-acm-cloudformation/contracts/`  
**参照**: `specs/001-add-acm-cloudformation/spec.md` (FR-003)

## 概要

このディレクトリには、CloudFormation テンプレートの出力（Outputs）に対する契約定義（JSON Schema）を格納する。これらのスキーマは、CloudFormation スタックの出力が期待される形式に準拠していることを検証するために使用される。

## ファイル一覧

| ファイル | 説明 |
|----------|------|
| `outputs.schema.json` | ACM CloudFormation スタックの Outputs 契約定義 |

## outputs.schema.json

### 目的

`outputs.schema.json` は、ACM 証明書を作成する CloudFormation スタック（`cloudformation/acm.yaml`）の Outputs が期待される形式に準拠していることを検証するための JSON Schema である。

このスキーマを使用することで:

- CloudFormation テンプレートの Outputs が仕様（`spec.md` FR-003）に準拠していることを自動検証できる
- CI/CD パイプラインでの Outputs 形式の自動チェックが可能になる
- 運用者が期待する出力形式を明確に把握できる

### 期待される Outputs

CloudFormation スタックは以下のフィールドを Outputs として出力することを期待する:

| フィールド | 型 | 必須 | 説明 |
|------------|-----|:----:|------|
| `RecordName` | string | ✓ | DNS 検証用レコードの名前（例: `_abcde.example.com.`） |
| `RecordType` | string | ✓ | レコード種別（`CNAME` または `TXT`） |
| `RecordValue` | string | ✓ | 検証用レコードの値（CNAME のターゲットまたは TXT の文字列） |
| `DomainName` | string | ✓ | 検証対象のドメイン（例: `example.com`） |
| `CertificateArn` | string | ✓ | 作成された ACM 証明書の ARN |
| `TTL` | integer | - | レコードの TTL（秒）。運用上の可読性向上のためのオプション |
| `Description` | string | - | 人間向け説明。運用上の可読性向上のためのオプション |

### スキーマ定義

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ACM CloudFormation Outputs",
  "type": "object",
  "properties": {
    "RecordName": { "type": "string" },
    "RecordType": { "type": "string", "enum": ["CNAME", "TXT"] },
    "RecordValue": { "type": "string" },
    "DomainName": { "type": "string" },
    "CertificateArn": { "type": "string" },
    "TTL": { "type": "integer" },
    "Description": { "type": "string" }
  },
  "required": ["RecordName", "RecordType", "RecordValue", "DomainName", "CertificateArn"],
  "additionalProperties": false
}
```

## スキーマ検証手順

### 手動検証（AWS CLI）

CloudFormation スタックの Outputs を取得し、JSON 形式で検証ツールに渡すことができる。

```bash
# 1. CloudFormation スタックの Outputs を取得
aws cloudformation describe-stacks \
  --stack-name nagiyu-acm-example-com \
  --region us-east-1 \
  --query 'Stacks[0].Outputs' \
  --output json > /tmp/outputs.json

# 2. Outputs を検証用 JSON に変換
# CloudFormation の Outputs は配列形式のため、オブジェクト形式に変換が必要
jq 'map({(.OutputKey): .OutputValue}) | add' /tmp/outputs.json > /tmp/outputs_object.json

# 3. JSON Schema で検証（ajv-cli を使用する例）
npx ajv validate \
  -s specs/001-add-acm-cloudformation/contracts/outputs.schema.json \
  -d /tmp/outputs_object.json
```

### 自動検証スクリプト（予定）

将来的に `scripts/validate_outputs_schema.sh`（[タスク T017](../tasks.md)）を追加し、上記の手順を自動化する予定。

```bash
# 使用例（予定）
./scripts/validate_outputs_schema.sh --stack-name nagiyu-acm-example-com
```

### Python での検証例

```python
import json
import jsonschema

# スキーマをロード
with open('specs/001-add-acm-cloudformation/contracts/outputs.schema.json') as f:
    schema = json.load(f)

# 検証対象データ（CloudFormation Outputs から変換済み）
outputs = {
    "RecordName": "_abcde.example.com.",
    "RecordType": "CNAME",
    "RecordValue": "_xyz.acm-validations.aws.",
    "DomainName": "example.com",
    "CertificateArn": "arn:aws:acm:us-east-1:123456789012:certificate/abc123"
}

# 検証実行
try:
    jsonschema.validate(outputs, schema)
    print("✓ Outputs are valid")
except jsonschema.ValidationError as e:
    print(f"✗ Validation error: {e.message}")
```

## CloudFormation テンプレートでの実装

`cloudformation/acm.yaml` で Outputs を定義する際、このスキーマに準拠する形式で出力すること。

**注記**: 以下の例ではインデックス `0` を使用して最初のドメイン検証オプションを参照している。これは単一ドメイン（またはワイルドカードを含む）の証明書を前提としている。複数の異なるドメインを含む証明書の場合は、各ドメインに対応する検証オプションを個別に出力する必要がある。

```yaml
Outputs:
  RecordName:
    Description: DNS validation record name
    Value: !GetAtt Certificate.DomainValidationOptions.0.ResourceRecord.Name
  RecordType:
    Description: DNS validation record type
    Value: !GetAtt Certificate.DomainValidationOptions.0.ResourceRecord.Type
  RecordValue:
    Description: DNS validation record value
    Value: !GetAtt Certificate.DomainValidationOptions.0.ResourceRecord.Value
  DomainName:
    Description: Domain name for the certificate
    Value: !Ref RootDomain
  CertificateArn:
    Description: ARN of the ACM certificate
    Value: !Ref Certificate
```

## 関連ドキュメント

- [spec.md](../spec.md) - 機能仕様書（FR-003 で Outputs の必須フィールドを定義）
- [data-model.md](../data-model.md) - データモデル定義（DNS Record エンティティ）
- [quickstart.md](../quickstart.md) - クイックスタートガイド（Outputs の確認手順）
- [tasks.md](../tasks.md) - タスク一覧（T005: Outputs 実装、T017: 検証スクリプト）

## 参照

- [JSON Schema Draft-07](http://json-schema.org/draft-07/schema#)
- [AWS CloudFormation Outputs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/outputs-section-structure.html)
- [AWS ACM Certificate Resource](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-certificatemanager-certificate.html)
