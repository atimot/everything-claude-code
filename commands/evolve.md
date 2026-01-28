---
name: evolve
description: 関連するインスティンクトをスキル、コマンド、またはエージェントにクラスタリングします
command: /evolve
implementation: python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py evolve
---

# Evolveコマンド

## 実装

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py evolve [--generate]
```

インスティンクトを分析し、関連するものをより高レベルの構造にクラスタリングします:
- **コマンド**: インスティンクトがユーザーから呼び出されるアクションを記述している場合
- **スキル**: インスティンクトが自動的にトリガーされる振る舞いを記述している場合
- **エージェント**: インスティンクトが複雑なマルチステッププロセスを記述している場合

## 使い方

```
/evolve                    # 全インスティンクトを分析し、進化を提案
/evolve --domain testing   # testingドメインのインスティンクトのみを進化
/evolve --dry-run          # 作成せずに作成される内容を表示
/evolve --threshold 5      # クラスタリングに5つ以上の関連インスティンクトを要求
```

## 進化ルール

### -> コマンド（ユーザーが呼び出す）
インスティンクトがユーザーが明示的にリクエストするアクションを記述している場合:
- 「ユーザーが...を依頼した場合」に関する複数のインスティンクト
- 「新しいXを作成する場合」のようなトリガーを持つインスティンクト
- 繰り返し可能なシーケンスに従うインスティンクト

例:
- `new-table-step1`: 「データベーステーブルを追加する場合、マイグレーションを作成」
- `new-table-step2`: 「データベーステーブルを追加する場合、スキーマを更新」
- `new-table-step3`: 「データベーステーブルを追加する場合、型を再生成」

-> 作成されるもの: `/new-table` コマンド

### -> スキル（自動トリガー）
インスティンクトが自動的に発生すべき振る舞いを記述している場合:
- パターンマッチングトリガー
- エラーハンドリングレスポンス
- コードスタイルの強制

例:
- `prefer-functional`: 「関数を書く場合、関数型スタイルを優先」
- `use-immutable`: 「状態を変更する場合、イミュータブルパターンを使用」
- `avoid-classes`: 「モジュールを設計する場合、クラスベースの設計を避ける」

-> 作成されるもの: `functional-patterns` スキル

### -> エージェント（深さ/分離が必要）
インスティンクトが分離の恩恵を受ける複雑なマルチステッププロセスを記述している場合:
- デバッグワークフロー
- リファクタリングシーケンス
- 調査タスク

例:
- `debug-step1`: 「デバッグ時、まずログを確認」
- `debug-step2`: 「デバッグ時、失敗しているコンポーネントを分離」
- `debug-step3`: 「デバッグ時、最小限の再現を作成」
- `debug-step4`: 「デバッグ時、テストで修正を検証」

-> 作成されるもの: `debugger` エージェント

## 実行内容

1. `~/.claude/homunculus/instincts/` からすべてのインスティンクトを読み取り
2. インスティンクトを以下の基準でグループ化:
   - ドメインの類似性
   - トリガーパターンの重複
   - アクションシーケンスの関係性
3. 3つ以上の関連インスティンクトの各クラスタに対して:
   - 進化の種類（コマンド/スキル/エージェント）を決定
   - 適切なファイルを生成
   - `~/.claude/homunculus/evolved/{commands,skills,agents}/` に保存
4. 進化した構造をソースインスティンクトにリンク

## 出力フォーマット

```
進化分析
==================

進化の準備ができた3つのクラスタが見つかりました:

## クラスタ 1: データベースマイグレーションワークフロー
インスティンクト: new-table-migration, update-schema, regenerate-types
種類: コマンド
信頼度: 85%（12回の観察に基づく）

作成されるもの: /new-table コマンド
ファイル:
  - ~/.claude/homunculus/evolved/commands/new-table.md

## クラスタ 2: 関数型コードスタイル
インスティンクト: prefer-functional, use-immutable, avoid-classes, pure-functions
種類: スキル
信頼度: 78%（8回の観察に基づく）

作成されるもの: functional-patterns スキル
ファイル:
  - ~/.claude/homunculus/evolved/skills/functional-patterns.md

## クラスタ 3: デバッグプロセス
インスティンクト: debug-check-logs, debug-isolate, debug-reproduce, debug-verify
種類: エージェント
信頼度: 72%（6回の観察に基づく）

作成されるもの: debugger エージェント
ファイル:
  - ~/.claude/homunculus/evolved/agents/debugger.md

---
`/evolve --execute` を実行してこれらのファイルを作成してください。
```

## フラグ

- `--execute`: 実際に進化した構造を作成（デフォルトはプレビュー）
- `--dry-run`: 作成せずにプレビュー
- `--domain <name>`: 指定したドメインのインスティンクトのみを進化
- `--threshold <n>`: クラスタを形成するために必要な最小インスティンクト数（デフォルト: 3）
- `--type <command|skill|agent>`: 指定した種類のみを作成

## 生成されるファイルのフォーマット

### コマンド
```markdown
---
name: new-table
description: Create a new database table with migration, schema update, and type generation
command: /new-table
evolved_from:
  - new-table-migration
  - update-schema
  - regenerate-types
---

# New Table Command

[クラスタリングされたインスティンクトに基づいて生成されたコンテンツ]

## ステップ
1. ...
2. ...
```

### スキル
```markdown
---
name: functional-patterns
description: Enforce functional programming patterns
evolved_from:
  - prefer-functional
  - use-immutable
  - avoid-classes
---

# Functional Patterns Skill

[クラスタリングされたインスティンクトに基づいて生成されたコンテンツ]
```

### エージェント
```markdown
---
name: debugger
description: Systematic debugging agent
model: sonnet
evolved_from:
  - debug-check-logs
  - debug-isolate
  - debug-reproduce
---

# Debugger Agent

[クラスタリングされたインスティンクトに基づいて生成されたコンテンツ]
```
