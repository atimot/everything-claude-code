---
name: tdd-workflow
description: 新機能の開発、バグ修正、リファクタリング時に使用するスキルです。ユニットテスト、インテグレーションテスト、E2Eテストを含む80%以上のカバレッジでテスト駆動開発を実施します。
---

# テスト駆動開発ワークフロー

このスキルは、すべてのコード開発がTDD原則に従い、包括的なテストカバレッジを確保します。

## 使用タイミング

- 新機能や機能の開発時
- バグや問題の修正時
- 既存コードのリファクタリング時
- APIエンドポイントの追加時
- 新しいコンポーネントの作成時

## 基本原則

### 1. コードの前にテストを書く
常にテストを先に書き、そのテストを通過するコードを実装します。

### 2. カバレッジ要件
- 最低80%のカバレッジ（ユニット + インテグレーション + E2E）
- すべてのエッジケースをカバー
- エラーシナリオのテスト
- 境界条件の検証

### 3. テストの種類

#### ユニットテスト
- 個別の関数やユーティリティ
- コンポーネントのロジック
- 純粋関数
- ヘルパーとユーティリティ

#### インテグレーションテスト
- APIエンドポイント
- データベース操作
- サービス間の連携
- 外部API呼び出し

#### E2Eテスト（Playwright）
- 重要なユーザーフロー
- 完全なワークフロー
- ブラウザ自動化
- UI操作

## TDDワークフローの手順

### ステップ1：ユーザージャーニーを書く
```
As a [role], I want to [action], so that [benefit]

Example:
As a user, I want to search for markets semantically,
so that I can find relevant markets even without exact keywords.
```

### ステップ2：テストケースを生成する
各ユーザージャーニーに対して、包括的なテストケースを作成します：

```typescript
describe('Semantic Search', () => {
  it('returns relevant markets for query', async () => {
    // Test implementation
  })

  it('handles empty query gracefully', async () => {
    // Test edge case
  })

  it('falls back to substring search when Redis unavailable', async () => {
    // Test fallback behavior
  })

  it('sorts results by similarity score', async () => {
    // Test sorting logic
  })
})
```

### ステップ3：テストを実行する（失敗するはず）
```bash
npm test
# Tests should fail - we haven't implemented yet
```

### ステップ4：コードを実装する
テストを通過するための最小限のコードを書きます：

```typescript
// Implementation guided by tests
export async function searchMarkets(query: string) {
  // Implementation here
}
```

### ステップ5：再度テストを実行する
```bash
npm test
# Tests should now pass
```

### ステップ6：リファクタリング
テストがグリーンの状態を維持しながらコード品質を改善します：
- 重複の除去
- 命名の改善
- パフォーマンスの最適化
- 可読性の向上

### ステップ7：カバレッジを確認する
```bash
npm run test:coverage
# Verify 80%+ coverage achieved
```

## テストパターン

### ユニットテストパターン（Jest/Vitest）
```typescript
import { render, screen, fireEvent } from '@testing-library/react'
import { Button } from './Button'

describe('Button Component', () => {
  it('renders with correct text', () => {
    render(<Button>Click me</Button>)
    expect(screen.getByText('Click me')).toBeInTheDocument()
  })

  it('calls onClick when clicked', () => {
    const handleClick = jest.fn()
    render(<Button onClick={handleClick}>Click</Button>)

    fireEvent.click(screen.getByRole('button'))

    expect(handleClick).toHaveBeenCalledTimes(1)
  })

  it('is disabled when disabled prop is true', () => {
    render(<Button disabled>Click</Button>)
    expect(screen.getByRole('button')).toBeDisabled()
  })
})
```

### APIインテグレーションテストパターン
```typescript
import { NextRequest } from 'next/server'
import { GET } from './route'

describe('GET /api/markets', () => {
  it('returns markets successfully', async () => {
    const request = new NextRequest('http://localhost/api/markets')
    const response = await GET(request)
    const data = await response.json()

    expect(response.status).toBe(200)
    expect(data.success).toBe(true)
    expect(Array.isArray(data.data)).toBe(true)
  })

  it('validates query parameters', async () => {
    const request = new NextRequest('http://localhost/api/markets?limit=invalid')
    const response = await GET(request)

    expect(response.status).toBe(400)
  })

  it('handles database errors gracefully', async () => {
    // Mock database failure
    const request = new NextRequest('http://localhost/api/markets')
    // Test error handling
  })
})
```

### E2Eテストパターン（Playwright）
```typescript
import { test, expect } from '@playwright/test'

test('user can search and filter markets', async ({ page }) => {
  // Navigate to markets page
  await page.goto('/')
  await page.click('a[href="/markets"]')

  // Verify page loaded
  await expect(page.locator('h1')).toContainText('Markets')

  // Search for markets
  await page.fill('input[placeholder="Search markets"]', 'election')

  // Wait for debounce and results
  await page.waitForTimeout(600)

  // Verify search results displayed
  const results = page.locator('[data-testid="market-card"]')
  await expect(results).toHaveCount(5, { timeout: 5000 })

  // Verify results contain search term
  const firstResult = results.first()
  await expect(firstResult).toContainText('election', { ignoreCase: true })

  // Filter by status
  await page.click('button:has-text("Active")')

  // Verify filtered results
  await expect(results).toHaveCount(3)
})

test('user can create a new market', async ({ page }) => {
  // Login first
  await page.goto('/creator-dashboard')

  // Fill market creation form
  await page.fill('input[name="name"]', 'Test Market')
  await page.fill('textarea[name="description"]', 'Test description')
  await page.fill('input[name="endDate"]', '2025-12-31')

  // Submit form
  await page.click('button[type="submit"]')

  // Verify success message
  await expect(page.locator('text=Market created successfully')).toBeVisible()

  // Verify redirect to market page
  await expect(page).toHaveURL(/\/markets\/test-market/)
})
```

## テストファイルの構成

```
src/
├── components/
│   ├── Button/
│   │   ├── Button.tsx
│   │   ├── Button.test.tsx          # Unit tests
│   │   └── Button.stories.tsx       # Storybook
│   └── MarketCard/
│       ├── MarketCard.tsx
│       └── MarketCard.test.tsx
├── app/
│   └── api/
│       └── markets/
│           ├── route.ts
│           └── route.test.ts         # Integration tests
└── e2e/
    ├── markets.spec.ts               # E2E tests
    ├── trading.spec.ts
    └── auth.spec.ts
```

## 外部サービスのモック

### Supabaseモック
```typescript
jest.mock('@/lib/supabase', () => ({
  supabase: {
    from: jest.fn(() => ({
      select: jest.fn(() => ({
        eq: jest.fn(() => Promise.resolve({
          data: [{ id: 1, name: 'Test Market' }],
          error: null
        }))
      }))
    }))
  }
}))
```

### Redisモック
```typescript
jest.mock('@/lib/redis', () => ({
  searchMarketsByVector: jest.fn(() => Promise.resolve([
    { slug: 'test-market', similarity_score: 0.95 }
  ])),
  checkRedisHealth: jest.fn(() => Promise.resolve({ connected: true }))
}))
```

### OpenAIモック
```typescript
jest.mock('@/lib/openai', () => ({
  generateEmbedding: jest.fn(() => Promise.resolve(
    new Array(1536).fill(0.1) // Mock 1536-dim embedding
  ))
}))
```

## テストカバレッジの検証

### カバレッジレポートの実行
```bash
npm run test:coverage
```

### カバレッジ閾値
```json
{
  "jest": {
    "coverageThresholds": {
      "global": {
        "branches": 80,
        "functions": 80,
        "lines": 80,
        "statements": 80
      }
    }
  }
}
```

## よくあるテストの間違い

### 間違い：実装の詳細をテストする
```typescript
// Don't test internal state
expect(component.state.count).toBe(5)
```

### 正しい方法：ユーザーに見える振る舞いをテストする
```typescript
// Test what users see
expect(screen.getByText('Count: 5')).toBeInTheDocument()
```

### 間違い：壊れやすいセレクタ
```typescript
// Breaks easily
await page.click('.css-class-xyz')
```

### 正しい方法：セマンティックなセレクタ
```typescript
// Resilient to changes
await page.click('button:has-text("Submit")')
await page.click('[data-testid="submit-button"]')
```

### 間違い：テストの独立性がない
```typescript
// Tests depend on each other
test('creates user', () => { /* ... */ })
test('updates same user', () => { /* depends on previous test */ })
```

### 正しい方法：独立したテスト
```typescript
// Each test sets up its own data
test('creates user', () => {
  const user = createTestUser()
  // Test logic
})

test('updates user', () => {
  const user = createTestUser()
  // Update logic
})
```

## 継続的テスト

### 開発中のウォッチモード
```bash
npm test -- --watch
# Tests run automatically on file changes
```

### プリコミットフック
```bash
# Runs before every commit
npm test && npm run lint
```

### CI/CDとの統合
```yaml
# GitHub Actions
- name: Run Tests
  run: npm test -- --coverage
- name: Upload Coverage
  uses: codecov/codecov-action@v3
```

## ベストプラクティス

1. **テストを先に書く** - 常にTDDで
2. **1テストにつき1アサーション** - 単一の振る舞いに集中する
3. **わかりやすいテスト名** - テスト対象を明確に説明する
4. **Arrange-Act-Assert** - 明確なテスト構造
5. **外部依存関係をモックする** - ユニットテストを独立させる
6. **エッジケースをテストする** - null、undefined、空、大きな値
7. **エラーパスをテストする** - ハッピーパスだけでなく
8. **テストを高速に保つ** - ユニットテストは各50ms以内
9. **テスト後にクリーンアップ** - 副作用を残さない
10. **カバレッジレポートを確認する** - ギャップを特定する

## 成功指標

- 80%以上のコードカバレッジを達成
- すべてのテストが合格（グリーン）
- スキップまたは無効化されたテストがない
- テスト実行が高速（ユニットテストは30秒以内）
- E2Eテストが重要なユーザーフローをカバー
- テストが本番環境の前にバグを検出

---

**忘れないでください**：テストはオプションではありません。テストは、自信を持ったリファクタリング、迅速な開発、そして本番環境の信頼性を実現するためのセーフティネットです。
