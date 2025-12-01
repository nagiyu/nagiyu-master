# data-model.md

## Entities

- **ACM Certificate**
    - Fields:
        - `CertificateArn` (string, ARN)
        - `DomainName` (string, ルートドメイン例: example.com)
        - `AlternativeNames` (array[string], 例: `*.example.com`)
        - `Status` (enum: PENDING_VALIDATION, ISSUED, FAILED, etc.)
        - `Region` (固定: `us-east-1`)
    - Validation rules:
        - `DomainName` は有効な FQDN であること
        - `AlternativeNames` はワイルドカードを含める場合、同一ルートに属すること推奨
    - State transitions:
        - 作成 -> PENDING_VALIDATION -> ISSUED/FAILED

- **CloudFormation Stack**
    - Fields:
        - `StackName` (string)
        - `StackId` (string)
        - `Status` (CREATE_IN_PROGRESS, CREATE_COMPLETE, UPDATE_COMPLETE, etc.)
        - `Outputs` (object - see Outputs contract)

- **DNS Record (検証用)**
    - Fields:
        - `RecordName` (string)
        - `RecordType` (enum: CNAME, TXT)
        - `RecordValue` (string)
        - `TTL` (integer, optional)
        - `Description` (string, optional)
    - Notes:
        - 外部DNS に手動で追加するための情報を含む。運用者はこれを使ってレコードを追加する。

- **GitHub Actions Secret**
    - Fields:
        - `ROOT_DOMAIN` (string, 例: example.com)
    - Notes:
        - Actions はこのシークレットを読み取り、CloudFormation テンプレートのパラメータとして渡す。
