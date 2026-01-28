---
description: GoのTDDワークフローを強制します。まずテーブル駆動テストを書き、次に実装します。go test -coverで80%以上のカバレッジを検証します。
---

# Go TDDコマンド

このコマンドは、Go慣用のテストパターンを使用して、テスト駆動開発の方法論を強制します。

## このコマンドの動作

1. **型/インターフェースの定義**: 最初に関数シグネチャをスキャフォールド
2. **テーブル駆動テストの作成**: 包括的なテストケースを作成（RED）
3. **テストの実行**: テストが正しい理由で失敗することを確認
4. **コードの実装**: テストを通すための最小限のコードを記述（GREEN）
5. **リファクタリング**: テストをグリーンに保ちながら改善
6. **カバレッジの確認**: 80%以上のカバレッジを確保

## 使用するタイミング

`/go-test` は以下の場合に使用してください:
- 新しいGo関数を実装する場合
- 既存コードにテストカバレッジを追加する場合
- バグ修正時（まず失敗するテストを書く）
- 重要なビジネスロジックを構築する場合
- GoでのTDDワークフローを学習する場合

## TDDサイクル

```
RED     → 失敗するテーブル駆動テストを書く
GREEN   → テストを通す最小限のコードを実装
REFACTOR → コードを改善し、テストはグリーンを維持
REPEAT  → 次のテストケースへ
```

## セッション例

```text
User: /go-test メールアドレスを検証する関数が必要です

Agent:
# TDDセッション: メールバリデーター

## ステップ 1: インターフェースの定義

```go
// validator/email.go
package validator

// ValidateEmail は指定された文字列が有効なメールアドレスかどうかを検査します。
// 有効な場合はnilを返し、それ以外は問題を説明するエラーを返します。
func ValidateEmail(email string) error {
    panic("not implemented")
}
```

## ステップ 2: テーブル駆動テストの作成（RED）

```go
// validator/email_test.go
package validator

import (
    "testing"
)

func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name    string
        email   string
        wantErr bool
    }{
        // 有効なメールアドレス
        {"simple email", "user@example.com", false},
        {"with subdomain", "user@mail.example.com", false},
        {"with plus", "user+tag@example.com", false},
        {"with dots", "first.last@example.com", false},

        // 無効なメールアドレス
        {"empty string", "", true},
        {"no at sign", "userexample.com", true},
        {"no domain", "user@", true},
        {"no local part", "@example.com", true},
        {"double at", "user@@example.com", true},
        {"spaces", "user @example.com", true},
        {"no tld", "user@example", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := ValidateEmail(tt.email)
            if tt.wantErr && err == nil {
                t.Errorf("ValidateEmail(%q) = nil; want error", tt.email)
            }
            if !tt.wantErr && err != nil {
                t.Errorf("ValidateEmail(%q) = %v; want nil", tt.email, err)
            }
        })
    }
}
```

## ステップ 3: テスト実行 - 失敗を確認

```bash
$ go test ./validator/...

--- FAIL: TestValidateEmail (0.00s)
    --- FAIL: TestValidateEmail/simple_email (0.00s)
        panic: not implemented

FAIL
```

✓ 期待通りテストが失敗しました（panic）。

## ステップ 4: 最小限のコードを実装（GREEN）

```go
// validator/email.go
package validator

import (
    "errors"
    "regexp"
)

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

var (
    ErrEmailEmpty   = errors.New("email cannot be empty")
    ErrEmailInvalid = errors.New("email format is invalid")
)

func ValidateEmail(email string) error {
    if email == "" {
        return ErrEmailEmpty
    }
    if !emailRegex.MatchString(email) {
        return ErrEmailInvalid
    }
    return nil
}
```

## ステップ 5: テスト実行 - パスを確認

```bash
$ go test ./validator/...

PASS
ok      project/validator    0.003s
```

✓ 全テストがパスしました！

## ステップ 6: カバレッジの確認

```bash
$ go test -cover ./validator/...

PASS
coverage: 100.0% of statements
ok      project/validator    0.003s
```

✓ カバレッジ: 100%

## TDD完了！
```

## テストパターン

### テーブル駆動テスト
```go
tests := []struct {
    name     string
    input    InputType
    want     OutputType
    wantErr  bool
}{
    {"case 1", input1, want1, false},
    {"case 2", input2, want2, true},
}

for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got, err := Function(tt.input)
        // アサーション
    })
}
```

### 並列テスト
```go
for _, tt := range tests {
    tt := tt // キャプチャ
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        // テスト本体
    })
}
```

### テストヘルパー
```go
func setupTestDB(t *testing.T) *sql.DB {
    t.Helper()
    db := createDB()
    t.Cleanup(func() { db.Close() })
    return db
}
```

## カバレッジコマンド

```bash
# 基本カバレッジ
go test -cover ./...

# カバレッジプロファイル
go test -coverprofile=coverage.out ./...

# ブラウザで表示
go tool cover -html=coverage.out

# 関数別カバレッジ
go tool cover -func=coverage.out

# 競合検出付き
go test -race -cover ./...
```

## カバレッジ目標

| コードの種類 | 目標値 |
|--------------|--------|
| 重要なビジネスロジック | 100% |
| パブリックAPI | 90%以上 |
| 一般的なコード | 80%以上 |
| 生成コード | 除外 |

## TDDベストプラクティス

**やるべきこと:**
- 実装の前にまずテストを書く
- 変更のたびにテストを実行する
- 包括的なカバレッジのためにテーブル駆動テストを使用する
- 実装の詳細ではなく、振る舞いをテストする
- エッジケースを含める（空、nil、最大値）

**やってはいけないこと:**
- テストの前に実装を書く
- REDフェーズをスキップする
- プライベート関数を直接テストする
- テストで `time.Sleep` を使用する
- フレーキーなテストを放置する

## 関連コマンド

- `/go-build` - ビルドエラーの修正
- `/go-review` - 実装後のコードレビュー
- `/verify` - 完全な検証ループの実行

## 関連ファイル

- スキル: `skills/golang-testing/`
- スキル: `skills/tdd-workflow/`
