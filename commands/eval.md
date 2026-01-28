# Evalコマンド

eval駆動開発ワークフローを管理します。

## 使い方

`/eval [define|check|report|list] [feature-name]`

## Evalの定義

`/eval define feature-name`

新しいeval定義を作成:

1. `.claude/evals/feature-name.md` をテンプレートで作成:

```markdown
## EVAL: feature-name
作成日: $(date)

### 機能Eval
- [ ] [機能1の説明]
- [ ] [機能2の説明]

### 回帰Eval
- [ ] [既存の動作1が引き続き動作する]
- [ ] [既存の動作2が引き続き動作する]

### 成功基準
- 機能evalのpass@3 > 90%
- 回帰evalのpass^3 = 100%
```

2. ユーザーに具体的な基準の入力を促す

## Evalのチェック

`/eval check feature-name`

機能のevalを実行:

1. `.claude/evals/feature-name.md` からeval定義を読み取り
2. 各機能evalに対して:
   - 基準の検証を試みる
   - 合格/不合格を記録
   - 試行を `.claude/evals/feature-name.log` に記録
3. 各回帰evalに対して:
   - 関連するテストを実行
   - ベースラインと比較
   - 合格/不合格を記録
4. 現在のステータスを報告:

```
EVALチェック: feature-name
========================
機能: X/Y 合格
回帰: X/Y 合格
ステータス: 進行中 / 準備完了
```

## Evalレポート

`/eval report feature-name`

包括的なevalレポートを生成:

```
EVALレポート: feature-name
=========================
生成日: $(date)

機能EVAL
----------------
[eval-1]: 合格 (pass@1)
[eval-2]: 合格 (pass@2) - リトライが必要だった
[eval-3]: 不合格 - 備考を参照

回帰EVAL
----------------
[test-1]: 合格
[test-2]: 合格
[test-3]: 合格

メトリクス
-------
機能 pass@1: 67%
機能 pass@3: 100%
回帰 pass^3: 100%

備考
-----
[問題、エッジケース、所見など]

推奨事項
--------------
[出荷可能 / 追加作業が必要 / ブロック]
```

## Eval一覧

`/eval list`

すべてのeval定義を表示:

```
EVAL定義一覧
================
feature-auth      [3/5 合格] 進行中
feature-search    [5/5 合格] 準備完了
feature-export    [0/4 合格] 未着手
```

## 引数

$ARGUMENTS:
- `define <name>` - 新しいeval定義を作成
- `check <name>` - evalを実行してチェック
- `report <name>` - 完全なレポートを生成
- `list` - すべてのevalを表示
- `clean` - 古いevalログを削除（直近10回分を保持）
