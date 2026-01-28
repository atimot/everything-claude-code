# オーケストレーションコマンド

複雑なタスクのためのシーケンシャルエージェントワークフロー。

## 使い方

`/orchestrate [workflow-type] [task-description]`

## ワークフロータイプ

### feature
フル機能実装ワークフロー：
```
planner -> tdd-guide -> code-reviewer -> security-reviewer
```

### bugfix
バグ調査と修正ワークフロー：
```
explorer -> tdd-guide -> code-reviewer
```

### refactor
安全なリファクタリングワークフロー：
```
architect -> code-reviewer -> tdd-guide
```

### security
セキュリティ重視のレビュー：
```
security-reviewer -> code-reviewer -> architect
```

## 実行パターン

ワークフロー内の各エージェントに対して：

1. **エージェントを呼び出す** - 前のエージェントからのコンテキストを渡す
2. **出力を収集する** - 構造化された引き継ぎドキュメントとして
3. **次のエージェントに渡す** - チェーン内の次のエージェントへ
4. **結果を集約する** - 最終レポートにまとめる

## 引き継ぎドキュメントの形式

エージェント間で引き継ぎドキュメントを作成します：

```markdown
## HANDOFF: [previous-agent] -> [next-agent]

### コンテキスト
[実施した内容の要約]

### 発見事項
[主要な発見または決定事項]

### 変更されたファイル
[変更したファイルの一覧]

### 未解決の質問
[次のエージェントに向けた未解決事項]

### 推奨事項
[次のステップの提案]
```

## 例：機能実装ワークフロー

```
/orchestrate feature "Add user authentication"
```

実行内容：

1. **プランナーエージェント**
   - 要件を分析する
   - 実装計画を作成する
   - 依存関係を特定する
   - 出力: `HANDOFF: planner -> tdd-guide`

2. **TDDガイドエージェント**
   - プランナーの引き継ぎを読む
   - テストを先に書く
   - テストが通るように実装する
   - 出力: `HANDOFF: tdd-guide -> code-reviewer`

3. **コードレビューエージェント**
   - 実装をレビューする
   - 問題点をチェックする
   - 改善点を提案する
   - 出力: `HANDOFF: code-reviewer -> security-reviewer`

4. **セキュリティレビューエージェント**
   - セキュリティ監査
   - 脆弱性チェック
   - 最終承認
   - 出力: 最終レポート

## 最終レポートの形式

```
ORCHESTRATION REPORT
====================
Workflow: feature
Task: Add user authentication
Agents: planner -> tdd-guide -> code-reviewer -> security-reviewer

SUMMARY
-------
[1段落の要約]

AGENT OUTPUTS
-------------
Planner: [要約]
TDD Guide: [要約]
Code Reviewer: [要約]
Security Reviewer: [要約]

FILES CHANGED
-------------
[変更された全ファイルの一覧]

TEST RESULTS
------------
[テスト成功/失敗の要約]

SECURITY STATUS
---------------
[セキュリティに関する発見事項]

RECOMMENDATION
--------------
[SHIP / NEEDS WORK / BLOCKED]
```

## 並列実行

独立したチェックの場合、エージェントを並列実行します：

```markdown
### 並列フェーズ
同時に実行：
- code-reviewer（品質）
- security-reviewer（セキュリティ）
- architect（設計）

### 結果のマージ
出力を1つのレポートに統合する
```

## 引数

$ARGUMENTS:
- `feature <description>` - フル機能実装ワークフロー
- `bugfix <description>` - バグ修正ワークフロー
- `refactor <description>` - リファクタリングワークフロー
- `security <description>` - セキュリティレビューワークフロー
- `custom <agents> <description>` - カスタムエージェントシーケンス

## カスタムワークフローの例

```
/orchestrate custom "architect,tdd-guide,code-reviewer" "Redesign caching layer"
```

## ヒント

1. **複雑な機能にはプランナーから始める**
2. **マージ前には必ずコードレビューを含める**
3. **認証/決済/個人情報にはセキュリティレビューを使う**
4. **引き継ぎは簡潔に** - 次のエージェントに必要な情報に焦点を当てる
5. **必要に応じてエージェント間で検証を実行する**
