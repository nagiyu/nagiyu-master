# quickstart.md

簡単に CloudFormation テンプレートを GitHub Actions でデプロイし、ACM 証明書を発行する手順。

前提:
- GitHub リポジトリに `ROOT_DOMAIN` シークレットを追加済み（例: `example.com`）
- AWS 資格情報（アクセスキー）が GitHub Secrets に設定されている

手順:

1. GitHub Actions ワークフロー (`.github/workflows/deploy-acm.yml`) を用意する。

2. ワークフローの主要ステップ（例）:

```bash
# 簡易フロー例（擬似）
# Checkout
# Configure AWS credentials (from secrets)
# Deploy CloudFormation template
aws cloudformation deploy --stack-name nagiyu-acm-${{ secrets.ROOT_DOMAIN }} --template-file cloudformation/acm.yaml --parameter-overrides RootDomain=${{ secrets.ROOT_DOMAIN }} --region us-east-1
```

3. ワークフローを実行すると、CloudFormation Outputs に検証用レコード情報が出力される。ワークフローのログまたは GitHub Action の出力から以下を確認する:
- `RecordName`
- `RecordType`
- `RecordValue`

4. 外部DNS の管理画面で、出力された検証レコードを追加する。

5. DNS が伝播した後、ACM 証明書の状態が `ISSUED` になることを確認する。確認には AWS CLI を利用できます:

```bash
aws acm describe-certificate --certificate-arn <CertificateArn> --region us-east-1
```

運用メモ:
- DNS 伝播・検証時間は外部要因のため目標（60 分）に含めない。
- 証明書発行後、この証明書ARN を CloudFront 等の配信サービスに使用する。
