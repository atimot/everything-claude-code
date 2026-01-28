# 検証ループスキル

Claude Codeセッションのための包括的な検証システムです。

## 使用タイミング

以下の場合にこのスキルを呼び出してください：
- 機能や重要なコード変更の完了後
- PR作成前
- 品質ゲートの通過を確認したい場合
- リファクタリング後

## 検証フェーズ

### フェーズ1：ビルド検証
```bash
# Check if project builds
npm run build 2>&1 | tail -20
# OR
pnpm build 2>&1 | tail -20
```

ビルドが失敗した場合は、続行する前に停止して修正してください。

### フェーズ2：型チェック
```bash
# TypeScript projects
npx tsc --noEmit 2>&1 | head -30

# Python projects
pyright . 2>&1 | head -30
```

すべての型エラーを報告します。重要なエラーは続行前に修正してください。

### フェーズ3：リントチェック
```bash
# JavaScript/TypeScript
npm run lint 2>&1 | head -30

# Python
ruff check . 2>&1 | head -30
```

### フェーズ4：テストスイート
```bash
# Run tests with coverage
npm run test -- --coverage 2>&1 | tail -50

# Check coverage threshold
# Target: 80% minimum
```

レポート内容：
- テスト総数：X
- 合格：X
- 失敗：X
- カバレッジ：X%

### フェーズ5：セキュリティスキャン
```bash
# Check for secrets
grep -rn "sk-" --include="*.ts" --include="*.js" . 2>/dev/null | head -10
grep -rn "api_key" --include="*.ts" --include="*.js" . 2>/dev/null | head -10

# Check for console.log
grep -rn "console.log" --include="*.ts" --include="*.tsx" src/ 2>/dev/null | head -10
```

### フェーズ6：差分レビュー
```bash
# Show what changed
git diff --stat
git diff HEAD~1 --name-only
```

各変更ファイルを以下の観点でレビューします：
- 意図しない変更
- エラーハンドリングの欠落
- 潜在的なエッジケース

## 出力フォーマット

すべてのフェーズを実行した後、検証レポートを生成します：

```
VERIFICATION REPORT
==================

Build:     [PASS/FAIL]
Types:     [PASS/FAIL] (X errors)
Lint:      [PASS/FAIL] (X warnings)
Tests:     [PASS/FAIL] (X/Y passed, Z% coverage)
Security:  [PASS/FAIL] (X issues)
Diff:      [X files changed]

Overall:   [READY/NOT READY] for PR

Issues to Fix:
1. ...
2. ...
```

## 継続モード

長時間のセッションでは、15分ごとまたは大きな変更の後に検証を実行します：

```markdown
メンタルチェックポイントを設定する：
- 各関数の完了後
- コンポーネントの完成後
- 次のタスクに移る前

実行: /verify
```

## フックとの統合

このスキルはPostToolUseフックを補完しますが、より深い検証を提供します。
フックは問題を即座にキャッチし、このスキルは包括的なレビューを提供します。
