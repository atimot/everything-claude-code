---
name: continuous-learning-v2
description: フックによるセッション観察、信頼度スコアリング付きアトミックインスティンクトの作成、スキル/コマンド/エージェントへの進化を行うインスティンクトベースの学習システム。
version: 2.0.0
---

# 継続的学習 v2 - インスティンクトベースアーキテクチャ

Claude Codeのセッションを、アトミックな「インスティンクト（直感的行動パターン）」を通じて再利用可能な知識に変換する高度な学習システムです。インスティンクトとは、信頼度スコアリング付きの小さな学習済み行動パターンです。

## v2の新機能

| 機能 | v1 | v2 |
|------|----|----|
| 観察 | Stopフック（セッション終了時） | PreToolUse/PostToolUse（100%信頼性） |
| 分析 | メインコンテキスト | バックグラウンドエージェント（Haiku） |
| 粒度 | 完全なスキル | アトミックな「インスティンクト」 |
| 信頼度 | なし | 0.3〜0.9の重み付き |
| 進化 | スキルに直接変換 | インスティンクト → クラスタ → スキル/コマンド/エージェント |
| 共有 | なし | インスティンクトのエクスポート/インポート |

## インスティンクトモデル

インスティンクトとは、小さな学習済み行動パターンです：

```yaml
---
id: prefer-functional-style
trigger: "when writing new functions"
confidence: 0.7
domain: "code-style"
source: "session-observation"
---

# Prefer Functional Style

## Action
Use functional patterns over classes when appropriate.

## Evidence
- Observed 5 instances of functional pattern preference
- User corrected class-based approach to functional on 2025-01-15
```

**特性：**
- **アトミック** — 1つのトリガー、1つのアクション
- **信頼度重み付き** — 0.3 = 暫定的、0.9 = ほぼ確実
- **ドメインタグ付き** — code-style、testing、git、debugging、workflowなど
- **エビデンスに基づく** — どの観察から生成されたかを追跡

## 仕組み

```
セッションのアクティビティ
      │
      │ フックがプロンプト＋ツール使用をキャプチャ（100%信頼性）
      ▼
┌─────────────────────────────────────────┐
│         observations.jsonl              │
│   (プロンプト、ツール呼び出し、結果)       │
└─────────────────────────────────────────┘
      │
      │ オブザーバーエージェントが読み取り（バックグラウンド、Haiku）
      ▼
┌─────────────────────────────────────────┐
│          パターン検出                     │
│   • ユーザーの修正 → インスティンクト      │
│   • エラー解決 → インスティンクト          │
│   • 繰り返しワークフロー → インスティンクト │
└─────────────────────────────────────────┘
      │
      │ 作成/更新
      ▼
┌─────────────────────────────────────────┐
│         instincts/personal/             │
│   • prefer-functional.md (0.7)          │
│   • always-test-first.md (0.9)          │
│   • use-zod-validation.md (0.6)         │
└─────────────────────────────────────────┘
      │
      │ /evolve がクラスタリング
      ▼
┌─────────────────────────────────────────┐
│              evolved/                   │
│   • commands/new-feature.md             │
│   • skills/testing-workflow.md          │
│   • agents/refactor-specialist.md       │
└─────────────────────────────────────────┘
```

## クイックスタート

### 1. 観察フックの有効化

`~/.claude/settings.json`に以下を追加してください：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/skills/continuous-learning-v2/hooks/observe.sh pre"
      }]
    }],
    "PostToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/skills/continuous-learning-v2/hooks/observe.sh post"
      }]
    }]
  }
}
```

### 2. ディレクトリ構造の初期化

```bash
mkdir -p ~/.claude/homunculus/{instincts/{personal,inherited},evolved/{agents,skills,commands}}
touch ~/.claude/homunculus/observations.jsonl
```

### 3. オブザーバーエージェントの実行（任意）

オブザーバーはバックグラウンドで観察を分析できます：

```bash
# バックグラウンドオブザーバーを起動
~/.claude/skills/continuous-learning-v2/agents/start-observer.sh
```

## コマンド

| コマンド | 説明 |
|----------|------|
| `/instinct-status` | 学習済みインスティンクトの一覧を信頼度とともに表示 |
| `/evolve` | 関連するインスティンクトをクラスタリングしてスキル/コマンドに進化 |
| `/instinct-export` | 共有用にインスティンクトをエクスポート |
| `/instinct-import <file>` | 他者からインスティンクトをインポート |

## 設定

`config.json`を編集してください：

```json
{
  "version": "2.0",
  "observation": {
    "enabled": true,
    "store_path": "~/.claude/homunculus/observations.jsonl",
    "max_file_size_mb": 10,
    "archive_after_days": 7
  },
  "instincts": {
    "personal_path": "~/.claude/homunculus/instincts/personal/",
    "inherited_path": "~/.claude/homunculus/instincts/inherited/",
    "min_confidence": 0.3,
    "auto_approve_threshold": 0.7,
    "confidence_decay_rate": 0.05
  },
  "observer": {
    "enabled": true,
    "model": "haiku",
    "run_interval_minutes": 5,
    "patterns_to_detect": [
      "user_corrections",
      "error_resolutions",
      "repeated_workflows",
      "tool_preferences"
    ]
  },
  "evolution": {
    "cluster_threshold": 3,
    "evolved_path": "~/.claude/homunculus/evolved/"
  }
}
```

## ファイル構造

```
~/.claude/homunculus/
├── identity.json           # プロフィール、技術レベル
├── observations.jsonl      # 現在のセッションの観察データ
├── observations.archive/   # 処理済みの観察データ
├── instincts/
│   ├── personal/           # 自動学習されたインスティンクト
│   └── inherited/          # 他者からインポートされたもの
└── evolved/
    ├── agents/             # 生成された専門エージェント
    ├── skills/             # 生成されたスキル
    └── commands/           # 生成されたコマンド
```

## スキルクリエイターとの連携

[スキルクリエイター GitHub App](https://skill-creator.app)を使用すると、以下の**両方**が生成されるようになりました：
- 従来のSKILL.mdファイル（後方互換性のため）
- インスティンクトコレクション（v2学習システム用）

リポジトリ分析から生成されたインスティンクトには`source: "repo-analysis"`が付与され、ソースリポジトリのURLが含まれます。

## 信頼度スコアリング

信頼度は時間とともに変化します：

| スコア | 意味 | 動作 |
|--------|------|------|
| 0.3 | 暫定的 | 提案されるが強制されない |
| 0.5 | 中程度 | 関連する場合に適用 |
| 0.7 | 強い | 自動承認で適用 |
| 0.9 | ほぼ確実 | コア動作として扱う |

**信頼度が上昇**する場合：
- パターンが繰り返し観察された
- ユーザーが提案された行動を修正しなかった
- 他のソースからの類似インスティンクトが一致した

**信頼度が低下**する場合：
- ユーザーが行動を明示的に修正した
- 長期間パターンが観察されなかった
- 矛盾するエビデンスが出現した

## なぜ観察にスキルではなくフックを使うのか？

> 「v1はスキルによる観察に依存していました。スキルは確率的であり、Claudeの判断に基づいて約50〜80%の確率でしか発火しません。」

フックは**100%**確実に、決定論的に発火します。これにより：
- すべてのツール呼び出しが観察される
- パターンの見逃しがない
- 網羅的な学習が可能

## 後方互換性

v2はv1と完全に互換性があります：
- 既存の`~/.claude/skills/learned/`スキルは引き続き動作
- Stopフックも引き続き実行（ただしv2にもデータを供給）
- 段階的な移行パス：両方を並行して実行可能

## プライバシー

- 観察データはマシン上に**ローカル**で保持
- エクスポートできるのは**インスティンクト**（パターン）のみ
- 実際のコードや会話内容は共有されない
- エクスポート対象はユーザーが制御

## 関連リンク

- [スキルクリエイター](https://skill-creator.app) - リポジトリ履歴からインスティンクトを生成
- [Homunculus](https://github.com/humanplane/homunculus) - v2アーキテクチャのインスピレーション元
- [詳細ガイド](https://x.com/affaanmustafa/status/2014040193557471352) - 継続的学習セクション

---

*インスティンクトベースの学習：一つ一つの観察を通じて、Claudeにあなたのパターンを教えます。*
