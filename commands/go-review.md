---
description: Go言語の慣用的パターン、並行処理の安全性、エラーハンドリング、セキュリティに関する包括的なコードレビュー。go-reviewerエージェントを呼び出します。
---

# Goコードレビュー

このコマンドは **go-reviewer** エージェントを呼び出し、Go固有の包括的なコードレビューを実施します。

## このコマンドの動作

1. **Go変更の特定**: `git diff` で変更された `.go` ファイルを検出
2. **静的解析の実行**: `go vet`、`staticcheck`、`golangci-lint` を実行
3. **セキュリティスキャン**: SQLインジェクション、コマンドインジェクション、競合状態を検査
4. **並行処理レビュー**: goroutineの安全性、チャネルの使用法、mutexパターンを分析
5. **Go慣用性チェック**: コードがGoの慣習とベストプラクティスに従っているか検証
6. **レポート生成**: 重大度別に問題を分類

## 使用するタイミング

`/go-review` は以下の場合に使用してください:
- Goコードを書いたり修正した後
- Goの変更をコミットする前
- GoコードのPull Requestをレビューする際
- 新しいGoコードベースにオンボーディングする際
- Go慣用パターンを学習する際

## レビューカテゴリ

### CRITICAL（必ず修正）
- SQL/コマンドインジェクションの脆弱性
- 同期なしの競合状態
- goroutineリーク
- ハードコードされた認証情報
- unsafeポインタの使用
- クリティカルパスでのエラー無視

### HIGH（修正すべき）
- コンテキスト付きエラーラッピングの不足
- エラー返却の代わりにpanicを使用
- Contextが伝播されていない
- バッファなしチャネルによるデッドロック
- インターフェース未実装エラー
- mutex保護の不足

### MEDIUM（検討事項）
- 非慣用的なコードパターン
- エクスポートされたシンボルのgodocコメント不足
- 非効率な文字列結合
- スライスの事前割り当てなし
- テーブル駆動テストの未使用

## 実行される自動チェック

```bash
# 静的解析
go vet ./...

# 高度なチェック（インストール済みの場合）
staticcheck ./...
golangci-lint run

# 競合検出
go build -race ./...

# セキュリティ脆弱性
govulncheck ./...
```

## 使用例

```text
User: /go-review

Agent:
# Goコードレビューレポート

## レビュー対象ファイル
- internal/handler/user.go（変更済み）
- internal/service/auth.go（変更済み）

## 静的解析結果
✓ go vet: 問題なし
✓ staticcheck: 問題なし

## 検出された問題

[CRITICAL] 競合状態
ファイル: internal/service/auth.go:45
問題: 共有マップが同期なしでアクセスされている
```go
var cache = map[string]*Session{}  // 並行アクセス！

func GetSession(id string) *Session {
    return cache[id]  // 競合状態
}
```
修正方法: sync.RWMutex または sync.Map を使用
```go
var (
    cache   = map[string]*Session{}
    cacheMu sync.RWMutex
)

func GetSession(id string) *Session {
    cacheMu.RLock()
    defer cacheMu.RUnlock()
    return cache[id]
}
```

[HIGH] エラーコンテキストの不足
ファイル: internal/handler/user.go:28
問題: コンテキストなしでエラーが返されている
```go
return err  // コンテキストなし
```
修正方法: コンテキストを付けてラップ
```go
return fmt.Errorf("get user %s: %w", userID, err)
```

## サマリー
- CRITICAL: 1
- HIGH: 1
- MEDIUM: 0

推奨: ❌ CRITICAL問題が修正されるまでマージをブロック
```

## 承認基準

| ステータス | 条件 |
|------------|------|
| ✅ 承認 | CRITICALまたはHIGHの問題なし |
| ⚠️ 警告 | MEDIUMの問題のみ（注意してマージ） |
| ❌ ブロック | CRITICALまたはHIGHの問題が検出された |

## 他のコマンドとの連携

- `/go-test` を先に使用してテストが通ることを確認
- `/go-build` をビルドエラーが発生した場合に使用
- `/go-review` をコミット前に使用
- `/code-review` をGo固有でない懸念事項に使用

## 関連ファイル

- エージェント: `agents/go-reviewer.md`
- スキル: `skills/golang-patterns/`、`skills/golang-testing/`
