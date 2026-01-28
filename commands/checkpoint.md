# チェックポイントコマンド

ワークフロー内でチェックポイントを作成または検証します。

## 使い方

`/checkpoint [create|verify|list] [name]`

## チェックポイントの作成

チェックポイントを作成する場合:

1. `/verify quick` を実行して現在の状態がクリーンであることを確認
2. チェックポイント名でgit stashまたはコミットを作成
3. チェックポイントを `.claude/checkpoints.log` に記録:

```bash
echo "$(date +%Y-%m-%d-%H:%M) | $CHECKPOINT_NAME | $(git rev-parse --short HEAD)" >> .claude/checkpoints.log
```

4. チェックポイント作成完了を報告

## チェックポイントの検証

チェックポイントに対して検証する場合:

1. ログからチェックポイントを読み取り
2. 現在の状態をチェックポイントと比較:
   - チェックポイント以降に追加されたファイル
   - チェックポイント以降に変更されたファイル
   - 現在とチェックポイント時点のテスト合格率
   - 現在とチェックポイント時点のカバレッジ

3. レポート:
```
チェックポイント比較: $NAME
============================
変更ファイル数: X
テスト: +Y 合格 / -Z 失敗
カバレッジ: +X% / -Y%
ビルド: [合格/失敗]
```

## チェックポイント一覧

以下の情報とともにすべてのチェックポイントを表示:
- 名前
- タイムスタンプ
- Git SHA
- ステータス（現在、遅れ、先行）

## ワークフロー

典型的なチェックポイントの流れ:

```
[開始] --> /checkpoint create "feature-start"
   |
[実装] --> /checkpoint create "core-done"
   |
[テスト] --> /checkpoint verify "core-done"
   |
[リファクタリング] --> /checkpoint create "refactor-done"
   |
[PR] --> /checkpoint verify "feature-start"
```

## 引数

$ARGUMENTS:
- `create <name>` - 名前付きチェックポイントを作成
- `verify <name>` - 名前付きチェックポイントに対して検証
- `list` - すべてのチェックポイントを表示
- `clear` - 古いチェックポイントを削除（直近5件を保持）
