---
name: instinct-status
description: 学習済みのすべてのインスティンクトとその信頼度を表示します
command: /instinct-status
implementation: python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py status
---

# インスティンクトステータスコマンド

学習済みのすべてのインスティンクトを信頼度スコア付きで、ドメイン別にグループ化して表示します。

## 実装

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py status
```

## 使用方法

```
/instinct-status
/instinct-status --domain code-style
/instinct-status --low-confidence
```

## 実行内容

1. `~/.claude/homunculus/instincts/personal/` からすべてのインスティンクトファイルを読み込む
2. `~/.claude/homunculus/instincts/inherited/` から継承されたインスティンクトを読み込む
3. ドメイン別にグループ化し、信頼度バー付きで表示

## 出力形式

```
📊 インスティンクトステータス
==================

## コードスタイル（4件のインスティンクト）

### prefer-functional-style
トリガー: 新しい関数を書くとき
アクション: クラスよりも関数型パターンを使用
信頼度: ████████░░ 80%
ソース: session-observation | 最終更新: 2025-01-22

### use-path-aliases
トリガー: モジュールをインポートするとき
アクション: 相対インポートの代わりに@/パスエイリアスを使用
信頼度: ██████░░░░ 60%
ソース: repo-analysis (github.com/acme/webapp)

## テスト（2件のインスティンクト）

### test-first-workflow
トリガー: 新しい機能を追加するとき
アクション: まずテストを書き、次に実装
信頼度: █████████░ 90%
ソース: session-observation

## ワークフロー（3件のインスティンクト）

### grep-before-edit
トリガー: コードを変更するとき
アクション: Grepで検索し、Readで確認してからEdit
信頼度: ███████░░░ 70%
ソース: session-observation

---
合計: 9件のインスティンクト（個人4件、継承5件）
オブザーバー: 実行中（最終分析: 5分前）
```

## フラグ

- `--domain <name>`: ドメインでフィルタリング（code-style, testing, gitなど）
- `--low-confidence`: 信頼度が0.5未満のインスティンクトのみ表示
- `--high-confidence`: 信頼度が0.7以上のインスティンクトのみ表示
- `--source <type>`: ソースでフィルタリング（session-observation, repo-analysis, inherited）
- `--json`: プログラム的な使用のためにJSON形式で出力
