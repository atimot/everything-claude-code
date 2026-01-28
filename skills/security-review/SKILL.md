---
name: security-review
description: 認証の追加、ユーザー入力の処理、シークレットの管理、APIエンドポイントの作成、決済や機密性の高い機能を実装する際にこのスキルを使用してください。包括的なセキュリティチェックリストとパターンを提供します。
---

# セキュリティレビュースキル

このスキルは、すべてのコードがセキュリティのベストプラクティスに従い、潜在的な脆弱性を特定することを保証します。

## 有効化のタイミング

- 認証または認可の実装時
- ユーザー入力またはファイルアップロードの処理時
- 新しいAPIエンドポイントの作成時
- シークレットまたは認証情報の操作時
- 決済機能の実装時
- 機密データの保存または送信時
- サードパーティAPIの統合時

## セキュリティチェックリスト

### 1. シークレット管理

#### やってはいけないこと
```typescript
const apiKey = "sk-proj-xxxxx"  // ハードコードされたシークレット
const dbPassword = "password123" // ソースコード内
```

#### 常にやるべきこと
```typescript
const apiKey = process.env.OPENAI_API_KEY
const dbUrl = process.env.DATABASE_URL

// シークレットの存在を検証
if (!apiKey) {
  throw new Error('OPENAI_API_KEY not configured')
}
```

#### 検証手順
- [ ] ハードコードされたAPIキー、トークン、パスワードがないこと
- [ ] すべてのシークレットが環境変数に格納されていること
- [ ] `.env.local` が .gitignore に含まれていること
- [ ] gitの履歴にシークレットがないこと
- [ ] 本番シークレットがホスティングプラットフォーム（Vercel、Railway）に格納されていること

### 2. 入力バリデーション

#### ユーザー入力は常にバリデーションする
```typescript
import { z } from 'zod'

// バリデーションスキーマの定義
const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  age: z.number().int().min(0).max(150)
})

// 処理前にバリデーション
export async function createUser(input: unknown) {
  try {
    const validated = CreateUserSchema.parse(input)
    return await db.users.create(validated)
  } catch (error) {
    if (error instanceof z.ZodError) {
      return { success: false, errors: error.errors }
    }
    throw error
  }
}
```

#### ファイルアップロードのバリデーション
```typescript
function validateFileUpload(file: File) {
  // サイズチェック（最大5MB）
  const maxSize = 5 * 1024 * 1024
  if (file.size > maxSize) {
    throw new Error('File too large (max 5MB)')
  }

  // タイプチェック
  const allowedTypes = ['image/jpeg', 'image/png', 'image/gif']
  if (!allowedTypes.includes(file.type)) {
    throw new Error('Invalid file type')
  }

  // 拡張子チェック
  const allowedExtensions = ['.jpg', '.jpeg', '.png', '.gif']
  const extension = file.name.toLowerCase().match(/\.[^.]+$/)?.[0]
  if (!extension || !allowedExtensions.includes(extension)) {
    throw new Error('Invalid file extension')
  }

  return true
}
```

#### 検証手順
- [ ] すべてのユーザー入力がスキーマでバリデーションされていること
- [ ] ファイルアップロードが制限されていること（サイズ、タイプ、拡張子）
- [ ] ユーザー入力がクエリに直接使用されていないこと
- [ ] ホワイトリストバリデーション（ブラックリストではない）
- [ ] エラーメッセージが機密情報を漏洩しないこと

### 3. SQLインジェクション防止

#### SQLの文字列結合は絶対にしない
```typescript
// 危険 - SQLインジェクションの脆弱性
const query = `SELECT * FROM users WHERE email = '${userEmail}'`
await db.query(query)
```

#### 常にパラメータ化クエリを使用する
```typescript
// 安全 - パラメータ化クエリ
const { data } = await supabase
  .from('users')
  .select('*')
  .eq('email', userEmail)

// または生SQLの場合
await db.query(
  'SELECT * FROM users WHERE email = $1',
  [userEmail]
)
```

#### 検証手順
- [ ] すべてのデータベースクエリがパラメータ化クエリを使用していること
- [ ] SQLで文字列結合が使用されていないこと
- [ ] ORM/クエリビルダーが正しく使用されていること
- [ ] Supabaseクエリが適切にサニタイズされていること

### 4. 認証と認可

#### JWTトークンの取り扱い
```typescript
// やってはいけない: localStorage（XSSに脆弱）
localStorage.setItem('token', token)

// 正しい方法: httpOnly Cookie
res.setHeader('Set-Cookie',
  `token=${token}; HttpOnly; Secure; SameSite=Strict; Max-Age=3600`)
```

#### 認可チェック
```typescript
export async function deleteUser(userId: string, requesterId: string) {
  // 常に最初に認可を検証する
  const requester = await db.users.findUnique({
    where: { id: requesterId }
  })

  if (requester.role !== 'admin') {
    return NextResponse.json(
      { error: 'Unauthorized' },
      { status: 403 }
    )
  }

  // 削除を実行
  await db.users.delete({ where: { id: userId } })
}
```

#### Row Level Security（Supabase）
```sql
-- すべてのテーブルでRLSを有効化
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- ユーザーは自分のデータのみ閲覧可能
CREATE POLICY "Users view own data"
  ON users FOR SELECT
  USING (auth.uid() = id);

-- ユーザーは自分のデータのみ更新可能
CREATE POLICY "Users update own data"
  ON users FOR UPDATE
  USING (auth.uid() = id);
```

#### 検証手順
- [ ] トークンがhttpOnly Cookieに格納されていること（localStorageではない）
- [ ] 機密操作前に認可チェックが行われていること
- [ ] SupabaseでRow Level Securityが有効化されていること
- [ ] ロールベースアクセス制御が実装されていること
- [ ] セッション管理が安全であること

### 5. XSS防止

#### HTMLのサニタイズ
```typescript
import DOMPurify from 'isomorphic-dompurify'

// ユーザー提供のHTMLは常にサニタイズする
function renderUserContent(html: string) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p'],
    ALLOWED_ATTR: []
  })
  return <div dangerouslySetInnerHTML={{ __html: clean }} />
}
```

#### コンテンツセキュリティポリシー
```typescript
// next.config.js
const securityHeaders = [
  {
    key: 'Content-Security-Policy',
    value: `
      default-src 'self';
      script-src 'self' 'unsafe-eval' 'unsafe-inline';
      style-src 'self' 'unsafe-inline';
      img-src 'self' data: https:;
      font-src 'self';
      connect-src 'self' https://api.example.com;
    `.replace(/\s{2,}/g, ' ').trim()
  }
]
```

#### 検証手順
- [ ] ユーザー提供のHTMLがサニタイズされていること
- [ ] CSPヘッダーが設定されていること
- [ ] バリデーションされていない動的コンテンツのレンダリングがないこと
- [ ] ReactのビルトインXSS保護が使用されていること

### 6. CSRF保護

#### CSRFトークン
```typescript
import { csrf } from '@/lib/csrf'

export async function POST(request: Request) {
  const token = request.headers.get('X-CSRF-Token')

  if (!csrf.verify(token)) {
    return NextResponse.json(
      { error: 'Invalid CSRF token' },
      { status: 403 }
    )
  }

  // リクエストを処理
}
```

#### SameSite Cookie
```typescript
res.setHeader('Set-Cookie',
  `session=${sessionId}; HttpOnly; Secure; SameSite=Strict`)
```

#### 検証手順
- [ ] 状態変更操作にCSRFトークンが設定されていること
- [ ] すべてのCookieにSameSite=Strictが設定されていること
- [ ] ダブルサブミットCookieパターンが実装されていること

### 7. レート制限

#### APIレート制限
```typescript
import rateLimit from 'express-rate-limit'

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15分
  max: 100, // ウィンドウあたり100リクエスト
  message: 'Too many requests'
})

// ルートに適用
app.use('/api/', limiter)
```

#### 高負荷な操作
```typescript
// 検索に対する厳格なレート制限
const searchLimiter = rateLimit({
  windowMs: 60 * 1000, // 1分
  max: 10, // 1分あたり10リクエスト
  message: 'Too many search requests'
})

app.use('/api/search', searchLimiter)
```

#### 検証手順
- [ ] すべてのAPIエンドポイントにレート制限が設定されていること
- [ ] 高負荷な操作にはより厳格な制限があること
- [ ] IPベースのレート制限が設定されていること
- [ ] ユーザーベースのレート制限（認証済み）が設定されていること

### 8. 機密データの漏洩

#### ログ出力
```typescript
// やってはいけない: 機密データのログ出力
console.log('User login:', { email, password })
console.log('Payment:', { cardNumber, cvv })

// 正しい方法: 機密データを隠蔽
console.log('User login:', { email, userId })
console.log('Payment:', { last4: card.last4, userId })
```

#### エラーメッセージ
```typescript
// やってはいけない: 内部詳細の公開
catch (error) {
  return NextResponse.json(
    { error: error.message, stack: error.stack },
    { status: 500 }
  )
}

// 正しい方法: 汎用的なエラーメッセージ
catch (error) {
  console.error('Internal error:', error)
  return NextResponse.json(
    { error: 'An error occurred. Please try again.' },
    { status: 500 }
  )
}
```

#### 検証手順
- [ ] ログにパスワード、トークン、シークレットが含まれていないこと
- [ ] ユーザーへのエラーメッセージが汎用的であること
- [ ] 詳細なエラーはサーバーログにのみ記録されていること
- [ ] スタックトレースがユーザーに公開されていないこと

### 9. ブロックチェーンセキュリティ（Solana）

#### ウォレット検証
```typescript
import { verify } from '@solana/web3.js'

async function verifyWalletOwnership(
  publicKey: string,
  signature: string,
  message: string
) {
  try {
    const isValid = verify(
      Buffer.from(message),
      Buffer.from(signature, 'base64'),
      Buffer.from(publicKey, 'base64')
    )
    return isValid
  } catch (error) {
    return false
  }
}
```

#### トランザクション検証
```typescript
async function verifyTransaction(transaction: Transaction) {
  // 受取人の検証
  if (transaction.to !== expectedRecipient) {
    throw new Error('Invalid recipient')
  }

  // 金額の検証
  if (transaction.amount > maxAmount) {
    throw new Error('Amount exceeds limit')
  }

  // ユーザーの残高が十分か検証
  const balance = await getBalance(transaction.from)
  if (balance < transaction.amount) {
    throw new Error('Insufficient balance')
  }

  return true
}
```

#### 検証手順
- [ ] ウォレット署名が検証されていること
- [ ] トランザクションの詳細がバリデーションされていること
- [ ] トランザクション前に残高チェックが行われていること
- [ ] ブラインドトランザクション署名がないこと

### 10. 依存関係のセキュリティ

#### 定期的な更新
```bash
# 脆弱性のチェック
npm audit

# 自動修正可能な問題を修正
npm audit fix

# 依存関係の更新
npm update

# 古いパッケージのチェック
npm outdated
```

#### ロックファイル
```bash
# ロックファイルは常にコミットする
git add package-lock.json

# CI/CDで再現可能なビルドのために使用
npm ci  # npm installの代わりに
```

#### 検証手順
- [ ] 依存関係が最新であること
- [ ] 既知の脆弱性がないこと（npm auditがクリーン）
- [ ] ロックファイルがコミットされていること
- [ ] GitHubでDependabotが有効化されていること
- [ ] 定期的なセキュリティ更新が行われていること

## セキュリティテスト

### 自動セキュリティテスト
```typescript
// 認証テスト
test('requires authentication', async () => {
  const response = await fetch('/api/protected')
  expect(response.status).toBe(401)
})

// 認可テスト
test('requires admin role', async () => {
  const response = await fetch('/api/admin', {
    headers: { Authorization: `Bearer ${userToken}` }
  })
  expect(response.status).toBe(403)
})

// 入力バリデーションテスト
test('rejects invalid input', async () => {
  const response = await fetch('/api/users', {
    method: 'POST',
    body: JSON.stringify({ email: 'not-an-email' })
  })
  expect(response.status).toBe(400)
})

// レート制限テスト
test('enforces rate limits', async () => {
  const requests = Array(101).fill(null).map(() =>
    fetch('/api/endpoint')
  )

  const responses = await Promise.all(requests)
  const tooManyRequests = responses.filter(r => r.status === 429)

  expect(tooManyRequests.length).toBeGreaterThan(0)
})
```

## デプロイ前セキュリティチェックリスト

本番デプロイ前に必ず確認：

- [ ] **シークレット**: ハードコードされたシークレットがなく、すべて環境変数に格納
- [ ] **入力バリデーション**: すべてのユーザー入力がバリデーション済み
- [ ] **SQLインジェクション**: すべてのクエリがパラメータ化済み
- [ ] **XSS**: ユーザーコンテンツがサニタイズ済み
- [ ] **CSRF**: 保護が有効化済み
- [ ] **認証**: 適切なトークン処理
- [ ] **認可**: ロールチェックが実施済み
- [ ] **レート制限**: すべてのエンドポイントで有効化済み
- [ ] **HTTPS**: 本番環境で強制済み
- [ ] **セキュリティヘッダー**: CSP、X-Frame-Optionsが設定済み
- [ ] **エラーハンドリング**: エラーに機密データが含まれていない
- [ ] **ログ**: 機密データがログに記録されていない
- [ ] **依存関係**: 最新で脆弱性がない
- [ ] **Row Level Security**: Supabaseで有効化済み
- [ ] **CORS**: 適切に設定済み
- [ ] **ファイルアップロード**: バリデーション済み（サイズ、タイプ）
- [ ] **ウォレット署名**: 検証済み（ブロックチェーンの場合）

## リソース

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Next.js セキュリティ](https://nextjs.org/docs/security)
- [Supabase セキュリティ](https://supabase.com/docs/guides/auth)
- [Web Security Academy](https://portswigger.net/web-security)

---

**注意**: セキュリティは任意ではありません。1つの脆弱性がプラットフォーム全体を危険にさらす可能性があります。迷った場合は、慎重な方を選んでください。
