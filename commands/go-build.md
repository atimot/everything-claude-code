---
description: Goのビルドエラー、go vet警告、リンター問題をインクリメンタルに修正します。最小限の外科的修正を行うgo-build-resolverエージェントを呼び出します。
---

# Goビルド＆修正

このコマンドは **go-build-resolver** エージェントを呼び出し、Goのビルドエラーを最小限の変更でインクリメンタルに修正します。

## このコマンドの動作

1. **診断の実行**: `go build`、`go vet`、`staticcheck` を実行
2. **エラーの解析**: ファイルごとにグループ化し、重大度順にソート
3. **インクリメンタルな修正**: 一度に1つのエラーを修正
4. **各修正の検証**: 変更のたびにビルドを再実行
5. **サマリーの報告**: 修正済みと未修正の内容を表示

## 使用するタイミング

`/go-build` は以下の場合に使用してください:
- `go build ./...` がエラーで失敗する場合
- `go vet ./...` が問題を報告する場合
- `golangci-lint run` が警告を表示する場合
- モジュールの依存関係が壊れている場合
- ビルドを壊す変更をプルした後

## 実行される診断コマンド

```bash
# プライマリビルドチェック
go build ./...

# 静的解析
go vet ./...

# 拡張リンティング（利用可能な場合）
staticcheck ./...
golangci-lint run

# モジュールの問題
go mod verify
go mod tidy -v
```

## セッション例

```text
User: /go-build

Agent:
# Goビルド解決

## 初期診断

```bash
$ go build ./...
internal/service/user.go:25:15: undefined: UserRepository
internal/handler/api.go:42:9: cannot use x (type string) as type int
internal/handler/api.go:58:2: missing return at end of function
```

エラー検出数: 3

## 修正 1: 未定義の識別子

ファイル: internal/service/user.go:25
エラー: undefined: UserRepository
原因: インポートの不足

```go
// インポートを追加
import "project/internal/repository"

// 変更前
var repo UserRepository
// 変更後
var repo repository.UserRepository
```

```bash
$ go build ./...
# 残りエラー: 2
```

## 修正 2: 型の不一致

ファイル: internal/handler/api.go:42
エラー: cannot use x (type string) as type int

```go
// 変更前
count := params.Get("count")
// 変更後
countStr := params.Get("count")
count, _ := strconv.Atoi(countStr)
```

```bash
$ go build ./...
# 残りエラー: 1
```

## 修正 3: return文の不足

ファイル: internal/handler/api.go:58
エラー: missing return at end of function

```go
func GetUser(id string) (*User, error) {
    if id == "" {
        return nil, ErrInvalidID
    }
    user := findUser(id)
    // 不足していたreturnを追加
    return user, nil
}
```

```bash
$ go build ./...
# ビルド成功！
```

## 最終検証

```bash
$ go vet ./...
# 問題なし

$ go test ./...
ok      project/internal/service   0.015s
ok      project/internal/handler   0.023s
```

## サマリー

| 指標 | 件数 |
|------|------|
| 修正したビルドエラー | 3 |
| 修正したvet警告 | 0 |
| 変更したファイル | 2 |
| 残りの問題 | 0 |

ビルドステータス: ✅ 成功
```

## よくあるエラーと修正

| エラー | 典型的な修正方法 |
|--------|------------------|
| `undefined: X` | インポートの追加またはタイポの修正 |
| `cannot use X as Y` | 型変換または代入の修正 |
| `missing return` | return文の追加 |
| `X does not implement Y` | 不足メソッドの追加 |
| `import cycle` | パッケージ構造の再編成 |
| `declared but not used` | 変数の削除または使用 |
| `cannot find package` | `go get` または `go mod tidy` |

## 修正戦略

1. **ビルドエラーを最優先** - コードがコンパイルできること
2. **vet警告を次に** - 疑わしい構文を修正
3. **リント警告を最後に** - スタイルとベストプラクティス
4. **一度に1つずつ修正** - 各変更を検証
5. **最小限の変更** - リファクタリングせず、修正のみ

## 停止条件

エージェントは以下の場合に停止して報告します:
- 同じエラーが3回の試行後も解消されない場合
- 修正がさらなるエラーを引き起こす場合
- アーキテクチャレベルの変更が必要な場合
- 外部依存関係が不足している場合

## 関連コマンド

- `/go-test` - ビルド成功後にテストを実行
- `/go-review` - コード品質をレビュー
- `/verify` - 完全な検証ループ

## 関連エージェント

このコマンドは `go-build-resolver` エージェントを呼び出します:
`agents/go-build-resolver.md`

関連スキル: `skills/golang-patterns/`
