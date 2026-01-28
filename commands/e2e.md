---
description: Playwrightを使用してE2Eテストを生成・実行します。テストジャーニーの作成、テストの実行、スクリーンショット/動画/トレースのキャプチャ、アーティファクトのアップロードを行います。
---

# E2Eコマンド

このコマンドは、Playwrightを使用してエンドツーエンドテストの生成、保守、実行を行う**e2e-runner**エージェントを呼び出します。

## このコマンドの機能

1. **テストジャーニーの生成** - ユーザーフローのPlaywrightテストを作成
2. **E2Eテストの実行** - 複数ブラウザ間でテストを実行
3. **アーティファクトのキャプチャ** - 失敗時にスクリーンショット、動画、トレースを取得
4. **結果のアップロード** - HTMLレポートとJUnit XML
5. **不安定なテストの検出** - 不安定なテストを隔離

## 使用タイミング

以下の場合に `/e2e` を使用してください:
- 重要なユーザージャーニーのテスト（ログイン、取引、決済）
- マルチステップフローがエンドツーエンドで機能するかの検証
- UIインタラクションとナビゲーションのテスト
- フロントエンドとバックエンド間の統合の検証
- 本番デプロイメントの準備

## 動作の仕組み

e2e-runnerエージェントは以下を行います:

1. **ユーザーフローを分析**し、テストシナリオを特定
2. Page Object Modelパターンを使用して**Playwrightテストを生成**
3. 複数ブラウザ（Chrome、Firefox、Safari）で**テストを実行**
4. スクリーンショット、動画、トレースで**失敗をキャプチャ**
5. 結果とアーティファクト付きの**レポートを生成**
6. **不安定なテストを特定**し、修正を推奨

## 使用例

```
User: /e2e マーケット検索と表示フローをテスト

Agent (e2e-runner):
# E2Eテスト生成: マーケット検索と表示フロー

## 特定されたテストシナリオ

**ユーザージャーニー:** マーケット検索 → 結果表示 → マーケットをクリック → 詳細表示

**テストカバレッジ:**
1. マーケットページに遷移
2. セマンティック検索を実行
3. 検索結果を検証
4. 最初の結果をクリック
5. マーケット詳細ページの読み込みを検証
6. チャートの描画を検証

## 生成されたテストコード

```typescript
// tests/e2e/markets/search-and-view.spec.ts
import { test, expect } from '@playwright/test'
import { MarketsPage } from '../../pages/MarketsPage'
import { MarketDetailsPage } from '../../pages/MarketDetailsPage'

test.describe('Market Search and View Flow', () => {
  test('user can search markets and view details', async ({ page }) => {
    // 1. Navigate to markets page
    const marketsPage = new MarketsPage(page)
    await marketsPage.goto()

    // Verify page loaded
    await expect(page).toHaveTitle(/Markets/)
    await expect(page.locator('h1')).toContainText('Markets')

    // 2. Perform semantic search
    await marketsPage.searchMarkets('election')

    // Wait for API response
    await page.waitForResponse(resp =>
      resp.url().includes('/api/markets/search') && resp.status() === 200
    )

    // 3. Verify search results
    const marketCards = marketsPage.marketCards
    await expect(marketCards.first()).toBeVisible()
    const resultCount = await marketCards.count()
    expect(resultCount).toBeGreaterThan(0)

    // Take screenshot of search results
    await page.screenshot({ path: 'artifacts/search-results.png' })

    // 4. Click on first result
    const firstMarketTitle = await marketCards.first().textContent()
    await marketCards.first().click()

    // 5. Verify market details page loads
    await expect(page).toHaveURL(/\/markets\/[a-z0-9-]+/)

    const detailsPage = new MarketDetailsPage(page)
    await expect(detailsPage.marketName).toBeVisible()
    await expect(detailsPage.marketDescription).toBeVisible()

    // 6. Verify chart renders
    await expect(detailsPage.priceChart).toBeVisible()

    // Verify market name matches
    const detailsTitle = await detailsPage.marketName.textContent()
    expect(detailsTitle?.toLowerCase()).toContain(
      firstMarketTitle?.toLowerCase().substring(0, 20) || ''
    )

    // Take screenshot of market details
    await page.screenshot({ path: 'artifacts/market-details.png' })
  })

  test('search with no results shows empty state', async ({ page }) => {
    const marketsPage = new MarketsPage(page)
    await marketsPage.goto()

    // Search for non-existent market
    await marketsPage.searchMarkets('xyznonexistentmarket123456')

    // Verify empty state
    await expect(page.locator('[data-testid="no-results"]')).toBeVisible()
    await expect(page.locator('[data-testid="no-results"]')).toContainText(
      /no.*results|no.*markets/i
    )

    const marketCount = await marketsPage.marketCards.count()
    expect(marketCount).toBe(0)
  })

  test('can clear search and see all markets again', async ({ page }) => {
    const marketsPage = new MarketsPage(page)
    await marketsPage.goto()

    // Initial market count
    const initialCount = await marketsPage.marketCards.count()

    // Perform search
    await marketsPage.searchMarkets('trump')
    await page.waitForLoadState('networkidle')

    // Verify filtered results
    const filteredCount = await marketsPage.marketCards.count()
    expect(filteredCount).toBeLessThan(initialCount)

    // Clear search
    await marketsPage.searchInput.clear()
    await page.waitForLoadState('networkidle')

    // Verify all markets shown again
    const finalCount = await marketsPage.marketCards.count()
    expect(finalCount).toBe(initialCount)
  })
})
```

## テストの実行

```bash
# 生成されたテストを実行
npx playwright test tests/e2e/markets/search-and-view.spec.ts

Running 3 tests using 3 workers

  ✓  [chromium] › search-and-view.spec.ts:5:3 › user can search markets and view details (4.2s)
  ✓  [chromium] › search-and-view.spec.ts:52:3 › search with no results shows empty state (1.8s)
  ✓  [chromium] › search-and-view.spec.ts:67:3 › can clear search and see all markets again (2.9s)

  3 passed (9.1s)

生成されたアーティファクト:
- artifacts/search-results.png
- artifacts/market-details.png
- playwright-report/index.html
```

## テストレポート

```
╔══════════════════════════════════════════════════════════════╗
║                    E2Eテスト結果                              ║
╠══════════════════════════════════════════════════════════════╣
║ ステータス: 全テスト合格                                       ║
║ 合計:       3テスト                                           ║
║ 合格:       3 (100%)                                         ║
║ 失敗:       0                                                ║
║ 不安定:     0                                                ║
║ 所要時間:   9.1秒                                             ║
╚══════════════════════════════════════════════════════════════╝

アーティファクト:
スクリーンショット: 2ファイル
動画: 0ファイル（失敗時のみ）
トレース: 0ファイル（失敗時のみ）
HTMLレポート: playwright-report/index.html

レポートを表示: npx playwright show-report
```

E2Eテストスイートのci/CD統合準備完了！
```

## テストアーティファクト

テスト実行時に以下のアーティファクトがキャプチャされます:

**全テスト共通:**
- タイムラインと結果を含むHTMLレポート
- CI統合用のJUnit XML

**失敗時のみ:**
- 失敗状態のスクリーンショット
- テストの動画記録
- デバッグ用トレースファイル（ステップごとのリプレイ）
- ネットワークログ
- コンソールログ

## アーティファクトの閲覧

```bash
# ブラウザでHTMLレポートを表示
npx playwright show-report

# 特定のトレースファイルを表示
npx playwright show-trace artifacts/trace-abc123.zip

# スクリーンショットはartifacts/ディレクトリに保存されます
open artifacts/search-results.png
```

## 不安定なテストの検出

テストが断続的に失敗する場合:

```
不安定なテストを検出: tests/e2e/markets/trade.spec.ts

テスト合格率 7/10回 (70%)

一般的な失敗原因:
"Timeout waiting for element '[data-testid="confirm-btn"]'"

推奨される修正:
1. 明示的な待機を追加: await page.waitForSelector('[data-testid="confirm-btn"]')
2. タイムアウトを延長: { timeout: 10000 }
3. コンポーネントの競合状態を確認
4. アニメーションによって要素が非表示になっていないか確認

隔離の推奨: 修正されるまでtest.fixme()としてマーク
```

## ブラウザ設定

デフォルトで複数のブラウザでテストが実行されます:
- Chromium（デスクトップChrome）
- Firefox（デスクトップ）
- WebKit（デスクトップSafari）
- モバイルChrome（オプション）

ブラウザの調整は `playwright.config.ts` で設定してください。

## CI/CD統合

CIパイプラインに追加:

```yaml
# .github/workflows/e2e.yml
- name: Install Playwright
  run: npx playwright install --with-deps

- name: Run E2E tests
  run: npx playwright test

- name: Upload artifacts
  if: always()
  uses: actions/upload-artifact@v3
  with:
    name: playwright-report
    path: playwright-report/
```

## PMX固有の重要フロー

PMXでは、以下のE2Eテストを優先してください:

**クリティカル（常に合格必須）:**
1. ユーザーがウォレットを接続できる
2. ユーザーがマーケットを閲覧できる
3. ユーザーがマーケットを検索できる（セマンティック検索）
4. ユーザーがマーケット詳細を表示できる
5. ユーザーが取引を実行できる（テスト資金で）
6. マーケットが正しく解決される
7. ユーザーが資金を引き出せる

**重要:**
1. マーケット作成フロー
2. ユーザープロフィールの更新
3. リアルタイム価格更新
4. チャートの描画
5. マーケットのフィルターとソート
6. モバイルレスポンシブレイアウト

## ベストプラクティス

**推奨:**
- 保守性のためにPage Object Modelを使用
- セレクターにはdata-testid属性を使用
- 任意のタイムアウトではなくAPIレスポンスを待機
- 重要なユーザージャーニーをエンドツーエンドでテスト
- mainへのマージ前にテストを実行
- テスト失敗時にアーティファクトを確認

**非推奨:**
- 脆いセレクターの使用（CSSクラスは変更される可能性がある）
- 実装の詳細をテスト
- 本番環境に対してテストを実行
- 不安定なテストを無視
- 失敗時のアーティファクト確認をスキップ
- すべてのエッジケースをE2Eでテスト（ユニットテストを使用）

## 重要な注意事項

**PMXにおけるクリティカル事項:**
- 実際のお金を含むE2Eテストはテストネット/ステージング環境でのみ実行すること
- 本番環境に対して取引テストを実行しないこと
- 金融テストには `test.skip(process.env.NODE_ENV === 'production')` を設定すること
- 少額のテスト資金のみを持つテストウォレットを使用すること

## 他のコマンドとの連携

- `/plan` を使用してテストすべき重要なジャーニーを特定
- `/tdd` をユニットテストに使用（より高速できめ細かい）
- `/e2e` を統合テストとユーザージャーニーテストに使用
- `/code-review` でテスト品質を検証

## 関連エージェント

このコマンドは以下にある `e2e-runner` エージェントを呼び出します:
`~/.claude/agents/e2e-runner.md`

## クイックコマンド

```bash
# 全E2Eテストを実行
npx playwright test

# 特定のテストファイルを実行
npx playwright test tests/e2e/markets/search.spec.ts

# ヘッドモードで実行（ブラウザを表示）
npx playwright test --headed

# テストをデバッグ
npx playwright test --debug

# テストコードを生成
npx playwright codegen http://localhost:3000

# レポートを表示
npx playwright show-report
```
