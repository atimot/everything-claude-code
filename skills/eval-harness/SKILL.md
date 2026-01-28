---
name: eval-harness
description: Claude Codeセッション向けの正式な評価フレームワーク。評価駆動開発（EDD）の原則を実装します。
tools: Read, Write, Edit, Bash, Grep, Glob
---

# 評価ハーネススキル

Claude Codeセッション向けの正式な評価フレームワークで、評価駆動開発（EDD）の原則を実装します。

## 理念

評価駆動開発は、評価を「AI開発のユニットテスト」として扱います：
- 実装の前に期待される振る舞いを定義する
- 開発中に評価を継続的に実行する
- 変更ごとにリグレッションを追跡する
- 信頼性測定にpass@kメトリクスを使用する

## 評価の種類

### 能力評価
Claudeが以前できなかったことができるようになったかをテストします：
```markdown
[CAPABILITY EVAL: feature-name]
Task: Description of what Claude should accomplish
Success Criteria:
  - [ ] Criterion 1
  - [ ] Criterion 2
  - [ ] Criterion 3
Expected Output: Description of expected result
```

### リグレッション評価
変更が既存の機能を壊していないことを確認します：
```markdown
[REGRESSION EVAL: feature-name]
Baseline: SHA or checkpoint name
Tests:
  - existing-test-1: PASS/FAIL
  - existing-test-2: PASS/FAIL
  - existing-test-3: PASS/FAIL
Result: X/Y passed (previously Y/Y)
```

## グレーダーの種類

### 1. コードベースのグレーダー
コードを使った決定論的なチェック：
```bash
# Check if file contains expected pattern
grep -q "export function handleAuth" src/auth.ts && echo "PASS" || echo "FAIL"

# Check if tests pass
npm test -- --testPathPattern="auth" && echo "PASS" || echo "FAIL"

# Check if build succeeds
npm run build && echo "PASS" || echo "FAIL"
```

### 2. モデルベースのグレーダー
Claudeを使ってオープンエンドの出力を評価します：
```markdown
[MODEL GRADER PROMPT]
Evaluate the following code change:
1. Does it solve the stated problem?
2. Is it well-structured?
3. Are edge cases handled?
4. Is error handling appropriate?

Score: 1-5 (1=poor, 5=excellent)
Reasoning: [explanation]
```

### 3. ヒューマングレーダー
手動レビュー用にフラグを立てます：
```markdown
[HUMAN REVIEW REQUIRED]
Change: Description of what changed
Reason: Why human review is needed
Risk Level: LOW/MEDIUM/HIGH
```

## メトリクス

### pass@k
「k回の試行で少なくとも1回成功」
- pass@1: 初回試行の成功率
- pass@3: 3回の試行以内の成功率
- 一般的な目標: pass@3 > 90%

### pass^k
「k回の試行すべてが成功」
- 信頼性に対するより高い基準
- pass^3: 3回連続の成功
- クリティカルパスに使用する

## 評価ワークフロー

### 1. 定義（コーディング前）
```markdown
## EVAL DEFINITION: feature-xyz

### Capability Evals
1. Can create new user account
2. Can validate email format
3. Can hash password securely

### Regression Evals
1. Existing login still works
2. Session management unchanged
3. Logout flow intact

### Success Metrics
- pass@3 > 90% for capability evals
- pass^3 = 100% for regression evals
```

### 2. 実装
定義された評価に合格するコードを書きます。

### 3. 評価
```bash
# Run capability evals
[Run each capability eval, record PASS/FAIL]

# Run regression evals
npm test -- --testPathPattern="existing"

# Generate report
```

### 4. レポート
```markdown
EVAL REPORT: feature-xyz
========================

Capability Evals:
  create-user:     PASS (pass@1)
  validate-email:  PASS (pass@2)
  hash-password:   PASS (pass@1)
  Overall:         3/3 passed

Regression Evals:
  login-flow:      PASS
  session-mgmt:    PASS
  logout-flow:     PASS
  Overall:         3/3 passed

Metrics:
  pass@1: 67% (2/3)
  pass@3: 100% (3/3)

Status: READY FOR REVIEW
```

## 統合パターン

### 実装前
```
/eval define feature-name
```
`.claude/evals/feature-name.md` に評価定義ファイルを作成します

### 実装中
```
/eval check feature-name
```
現在の評価を実行してステータスを報告します

### 実装後
```
/eval report feature-name
```
完全な評価レポートを生成します

## 評価の保存

プロジェクト内に評価を保存します：
```
.claude/
  evals/
    feature-xyz.md      # 評価定義
    feature-xyz.log     # 評価実行履歴
    baseline.json       # リグレッションベースライン
```

## ベストプラクティス

1. **コーディングの前に評価を定義する** - 成功基準を明確に考えることを強制します
2. **評価を頻繁に実行する** - リグレッションを早期に発見します
3. **pass@kを時系列で追跡する** - 信頼性のトレンドを監視します
4. **可能な限りコードグレーダーを使用する** - 決定論的 > 確率論的
5. **セキュリティにはヒューマンレビュー** - セキュリティチェックを完全に自動化しない
6. **評価を高速に保つ** - 遅い評価は実行されません
7. **評価をコードとともにバージョン管理する** - 評価はファーストクラスの成果物です

## 例：認証の追加

```markdown
## EVAL: add-authentication

### Phase 1: Define (10 min)
Capability Evals:
- [ ] User can register with email/password
- [ ] User can login with valid credentials
- [ ] Invalid credentials rejected with proper error
- [ ] Sessions persist across page reloads
- [ ] Logout clears session

Regression Evals:
- [ ] Public routes still accessible
- [ ] API responses unchanged
- [ ] Database schema compatible

### Phase 2: Implement (varies)
[Write code]

### Phase 3: Evaluate
Run: /eval check add-authentication

### Phase 4: Report
EVAL REPORT: add-authentication
==============================
Capability: 5/5 passed (pass@3: 100%)
Regression: 3/3 passed (pass^3: 100%)
Status: SHIP IT
```
