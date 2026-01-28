| name | description |
|------|-------------|
| cloud-infrastructure-security | クラウドプラットフォームへのデプロイ、インフラストラクチャの構成、IAMポリシーの管理、ログ/モニタリングの設定、CI/CDパイプラインの実装時にこのスキルを使用してください。ベストプラクティスに沿ったクラウドセキュリティチェックリストを提供します。 |

# クラウド＆インフラストラクチャセキュリティスキル

このスキルは、クラウドインフラストラクチャ、CI/CDパイプライン、デプロイ構成がセキュリティのベストプラクティスに従い、業界標準に準拠することを保証します。

## 有効化のタイミング

- クラウドプラットフォーム（AWS、Vercel、Railway、Cloudflare）へのアプリケーションデプロイ時
- IAMロールとパーミッションの設定時
- CI/CDパイプラインの設定時
- Infrastructure as Code（Terraform、CloudFormation）の実装時
- ログとモニタリングの設定時
- クラウド環境でのシークレット管理時
- CDNとエッジセキュリティの設定時
- 災害復旧とバックアップ戦略の実装時

## クラウドセキュリティチェックリスト

### 1. IAMとアクセス制御

#### 最小権限の原則

```yaml
# 正しい方法: 最小限のパーミッション
iam_role:
  permissions:
    - s3:GetObject  # 読み取りアクセスのみ
    - s3:ListBucket
  resources:
    - arn:aws:s3:::my-bucket/*  # 特定のバケットのみ

# やってはいけない: 過度に広いパーミッション
iam_role:
  permissions:
    - s3:*  # すべてのS3アクション
  resources:
    - "*"  # すべてのリソース
```

#### 多要素認証（MFA）

```bash
# ルート/管理者アカウントには常にMFAを有効化する
aws iam enable-mfa-device \
  --user-name admin \
  --serial-number arn:aws:iam::123456789:mfa/admin \
  --authentication-code1 123456 \
  --authentication-code2 789012
```

#### 検証手順

- [ ] 本番環境でルートアカウントが使用されていないこと
- [ ] すべての特権アカウントでMFAが有効化されていること
- [ ] サービスアカウントが長期認証情報ではなくロールを使用していること
- [ ] IAMポリシーが最小権限に従っていること
- [ ] 定期的なアクセスレビューが実施されていること
- [ ] 未使用の認証情報がローテーションまたは削除されていること

### 2. シークレット管理

#### クラウドシークレットマネージャー

```typescript
// 正しい方法: クラウドシークレットマネージャーを使用
import { SecretsManager } from '@aws-sdk/client-secrets-manager';

const client = new SecretsManager({ region: 'us-east-1' });
const secret = await client.getSecretValue({ SecretId: 'prod/api-key' });
const apiKey = JSON.parse(secret.SecretString).key;

// やってはいけない: ハードコードまたは環境変数のみ
const apiKey = process.env.API_KEY; // ローテーションされない、監査されない
```

#### シークレットのローテーション

```bash
# データベース認証情報の自動ローテーションを設定
aws secretsmanager rotate-secret \
  --secret-id prod/db-password \
  --rotation-lambda-arn arn:aws:lambda:region:account:function:rotate \
  --rotation-rules AutomaticallyAfterDays=30
```

#### 検証手順

- [ ] すべてのシークレットがクラウドシークレットマネージャー（AWS Secrets Manager、Vercel Secrets）に格納されていること
- [ ] データベース認証情報の自動ローテーションが有効化されていること
- [ ] APIキーが少なくとも四半期ごとにローテーションされていること
- [ ] コード、ログ、エラーメッセージにシークレットがないこと
- [ ] シークレットアクセスの監査ログが有効化されていること

### 3. ネットワークセキュリティ

#### VPCとファイアウォールの設定

```terraform
# 正しい方法: 制限されたセキュリティグループ
resource "aws_security_group" "app" {
  name = "app-sg"

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]  # 内部VPCのみ
  }

  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # HTTPSアウトバウンドのみ
  }
}

# やってはいけない: インターネットに公開
resource "aws_security_group" "bad" {
  ingress {
    from_port   = 0
    to_port     = 65535
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # すべてのポート、すべてのIP！
  }
}
```

#### 検証手順

- [ ] データベースが公開アクセス可能でないこと
- [ ] SSH/RDPポートがVPN/踏み台サーバーのみに制限されていること
- [ ] セキュリティグループが最小権限に従っていること
- [ ] ネットワークACLが設定されていること
- [ ] VPCフローログが有効化されていること

### 4. ログとモニタリング

#### CloudWatch/ログの設定

```typescript
// 正しい方法: 包括的なログ出力
import { CloudWatchLogsClient, CreateLogStreamCommand } from '@aws-sdk/client-cloudwatch-logs';

const logSecurityEvent = async (event: SecurityEvent) => {
  await cloudwatch.putLogEvents({
    logGroupName: '/aws/security/events',
    logStreamName: 'authentication',
    logEvents: [{
      timestamp: Date.now(),
      message: JSON.stringify({
        type: event.type,
        userId: event.userId,
        ip: event.ip,
        result: event.result,
        // 機密データは絶対にログに記録しない
      })
    }]
  });
};
```

#### 検証手順

- [ ] すべてのサービスでCloudWatch/ログが有効化されていること
- [ ] 認証失敗の試行がログに記録されていること
- [ ] 管理者の操作が監査されていること
- [ ] ログの保持期間が設定されていること（コンプライアンス要件で90日以上）
- [ ] 不審なアクティビティに対するアラートが設定されていること
- [ ] ログが一元管理され改ざん防止されていること

### 5. CI/CDパイプラインセキュリティ

#### セキュアなパイプライン設定

```yaml
# 正しい方法: セキュアなGitHub Actionsワークフロー
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read  # 最小限のパーミッション

    steps:
      - uses: actions/checkout@v4

      # シークレットのスキャン
      - name: Secret scanning
        uses: trufflesecurity/trufflehog@main

      # 依存関係の監査
      - name: Audit dependencies
        run: npm audit --audit-level=high

      # 長期トークンではなくOIDCを使用
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
          aws-region: us-east-1
```

#### サプライチェーンセキュリティ

```json
// package.json - ロックファイルと整合性チェックを使用
{
  "scripts": {
    "install": "npm ci",  // 再現可能なビルドのためにciを使用
    "audit": "npm audit --audit-level=moderate",
    "check": "npm outdated"
  }
}
```

#### 検証手順

- [ ] 長期認証情報の代わりにOIDCが使用されていること
- [ ] パイプラインでシークレットスキャンが実施されていること
- [ ] 依存関係の脆弱性スキャンが実施されていること
- [ ] コンテナイメージのスキャンが実施されていること（該当する場合）
- [ ] ブランチ保護ルールが適用されていること
- [ ] マージ前にコードレビューが必須であること
- [ ] 署名付きコミットが適用されていること

### 6. Cloudflare＆CDNセキュリティ

#### Cloudflareセキュリティ設定

```typescript
// 正しい方法: セキュリティヘッダー付きのCloudflare Workers
export default {
  async fetch(request: Request): Promise<Response> {
    const response = await fetch(request);

    // セキュリティヘッダーを追加
    const headers = new Headers(response.headers);
    headers.set('X-Frame-Options', 'DENY');
    headers.set('X-Content-Type-Options', 'nosniff');
    headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');
    headers.set('Permissions-Policy', 'geolocation=(), microphone=()');

    return new Response(response.body, {
      status: response.status,
      headers
    });
  }
};
```

#### WAFルール

```bash
# Cloudflare WAFマネージドルールを有効化
# - OWASP Core Ruleset
# - Cloudflare Managed Ruleset
# - レート制限ルール
# - ボット保護
```

#### 検証手順

- [ ] OWASPルールでWAFが有効化されていること
- [ ] レート制限が設定されていること
- [ ] ボット保護がアクティブであること
- [ ] DDoS保護が有効化されていること
- [ ] セキュリティヘッダーが設定されていること
- [ ] SSL/TLS厳格モードが有効化されていること

### 7. バックアップと災害復旧

#### 自動バックアップ

```terraform
# 正しい方法: 自動RDSバックアップ
resource "aws_db_instance" "main" {
  allocated_storage     = 20
  engine               = "postgres"

  backup_retention_period = 30  # 30日間の保持
  backup_window          = "03:00-04:00"
  maintenance_window     = "mon:04:00-mon:05:00"

  enabled_cloudwatch_logs_exports = ["postgresql"]

  deletion_protection = true  # 誤削除を防止
}
```

#### 検証手順

- [ ] 自動日次バックアップが設定されていること
- [ ] バックアップの保持期間がコンプライアンス要件を満たしていること
- [ ] ポイントインタイムリカバリが有効化されていること
- [ ] 四半期ごとにバックアップテストが実施されていること
- [ ] 災害復旧計画が文書化されていること
- [ ] RPOとRTOが定義されテストされていること

## デプロイ前クラウドセキュリティチェックリスト

本番クラウドデプロイ前に必ず確認：

- [ ] **IAM**: ルートアカウント未使用、MFA有効化、最小権限ポリシー
- [ ] **シークレット**: すべてのシークレットがクラウドシークレットマネージャーにローテーション付きで格納
- [ ] **ネットワーク**: セキュリティグループが制限され、公開データベースがない
- [ ] **ログ**: CloudWatch/ログが保持期間付きで有効化
- [ ] **モニタリング**: 異常に対するアラートが設定済み
- [ ] **CI/CD**: OIDC認証、シークレットスキャン、依存関係監査
- [ ] **CDN/WAF**: Cloudflare WAFがOWASPルールで有効化
- [ ] **暗号化**: データが保存時と転送時に暗号化
- [ ] **バックアップ**: テスト済みリカバリによる自動バックアップ
- [ ] **コンプライアンス**: GDPR/HIPAA要件への準拠（該当する場合）
- [ ] **ドキュメント**: インフラストラクチャが文書化され、ランブックが作成済み
- [ ] **インシデント対応**: セキュリティインシデント計画が策定済み

## よくあるクラウドセキュリティの設定ミス

### S3バケットの公開

```bash
# やってはいけない: パブリックバケット
aws s3api put-bucket-acl --bucket my-bucket --acl public-read

# 正しい方法: 特定のアクセス権を持つプライベートバケット
aws s3api put-bucket-acl --bucket my-bucket --acl private
aws s3api put-bucket-policy --bucket my-bucket --policy file://policy.json
```

### RDSのパブリックアクセス

```terraform
# やってはいけない
resource "aws_db_instance" "bad" {
  publicly_accessible = true  # 絶対にやらないこと！
}

# 正しい方法
resource "aws_db_instance" "good" {
  publicly_accessible = false
  vpc_security_group_ids = [aws_security_group.db.id]
}
```

## リソース

- [AWSセキュリティベストプラクティス](https://aws.amazon.com/security/best-practices/)
- [CIS AWS Foundations Benchmark](https://www.cisecurity.org/benchmark/amazon_web_services)
- [Cloudflareセキュリティドキュメント](https://developers.cloudflare.com/security/)
- [OWASPクラウドセキュリティ](https://owasp.org/www-project-cloud-security/)
- [Terraformセキュリティベストプラクティス](https://www.terraform.io/docs/cloud/guides/recommended-practices/)

**注意**: クラウドの設定ミスはデータ漏洩の最大の原因です。1つのS3バケットの公開や過度に寛容なIAMポリシーが、インフラストラクチャ全体を危険にさらす可能性があります。常に最小権限の原則と多層防御を遵守してください。
