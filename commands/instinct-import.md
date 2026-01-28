---
name: instinct-import
description: チームメイト、Skill Creator、または他のソースからインスティンクトをインポートします
command: /instinct-import
implementation: python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py import <file>
---

# インスティンクトインポートコマンド

## 実装

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py import <file-or-url> [--dry-run] [--force] [--min-confidence 0.7]
```

以下のソースからインスティンクトをインポートします:
- チームメイトのエクスポート
- Skill Creator（リポジトリ分析）
- コミュニティコレクション
- 以前のマシンのバックアップ

## 使用方法

```
/instinct-import team-instincts.yaml
/instinct-import https://github.com/org/repo/instincts.yaml
/instinct-import --from-skill-creator acme/webapp
```

## 実行内容

1. インスティンクトファイルを取得（ローカルパスまたはURL）
2. フォーマットを解析・検証
3. 既存のインスティンクトとの重複をチェック
4. 新しいインスティンクトをマージまたは追加
5. `~/.claude/homunculus/instincts/inherited/` に保存

## インポートプロセス

```
📥 インスティンクトをインポート中: team-instincts.yaml
================================================

インポート対象のインスティンクト: 12件

競合を分析中...

## 新規インスティンクト（8件）
以下が追加されます:
  ✓ use-zod-validation（信頼度: 0.7）
  ✓ prefer-named-exports（信頼度: 0.65）
  ✓ test-async-functions（信頼度: 0.8）
  ...

## 重複インスティンクト（3件）
類似のインスティンクトが既に存在:
  ⚠️ prefer-functional-style
     ローカル: 信頼度 0.8、観測回数 12
     インポート: 信頼度 0.7
     → ローカルを保持（信頼度が高い）

  ⚠️ test-first-workflow
     ローカル: 信頼度 0.75
     インポート: 信頼度 0.9
     → インポートに更新（信頼度が高い）

## 競合するインスティンクト（1件）
ローカルのインスティンクトと矛盾:
  ❌ use-classes-for-services
     競合先: avoid-classes
     → スキップ（手動での解決が必要）

---
新規8件をインポート、1件を更新、3件をスキップしますか？
```

## マージ戦略

### 重複の場合
既存のインスティンクトと一致するものをインポートする場合:
- **信頼度が高い方を優先**: 信頼度が高い方を保持
- **証拠のマージ**: 観測回数を合算
- **タイムスタンプの更新**: 最近検証されたものとしてマーク

### 競合の場合
既存のインスティンクトと矛盾するものをインポートする場合:
- **デフォルトでスキップ**: 競合するインスティンクトをインポートしない
- **レビュー対象としてフラグ**: 両方に注意が必要とマーク
- **手動解決**: ユーザーがどちらを保持するか決定

## ソースの追跡

インポートされたインスティンクトには以下がマークされます:
```yaml
source: "inherited"
imported_from: "team-instincts.yaml"
imported_at: "2025-01-22T10:30:00Z"
original_source: "session-observation"  # または "repo-analysis"
```

## Skill Creatorとの統合

Skill Creatorからインポートする場合:

```
/instinct-import --from-skill-creator acme/webapp
```

リポジトリ分析から生成されたインスティンクトを取得します:
- ソース: `repo-analysis`
- 初期信頼度が高い（0.7以上）
- ソースリポジトリにリンク

## フラグ

- `--dry-run`: インポートせずにプレビュー
- `--force`: 競合がある場合もインポート
- `--merge-strategy <higher|local|import>`: 重複時の処理方法
- `--from-skill-creator <owner/repo>`: Skill Creator分析からインポート
- `--min-confidence <n>`: しきい値以上のインスティンクトのみインポート

## 出力

インポート後:
```
✅ インポート完了！

追加: 8件のインスティンクト
更新: 1件のインスティンクト
スキップ: 3件のインスティンクト（重複2件、競合1件）

新しいインスティンクトの保存先: ~/.claude/homunculus/instincts/inherited/

/instinct-status を実行してすべてのインスティンクトを確認してください。
```
