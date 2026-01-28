---
name: continuous-learning
description: Claude Codeセッションから再利用可能なパターンを自動抽出し、学習済みスキルとして保存して将来の利用に備えます。
---

# 継続学習スキル

Claude Codeセッション終了時に自動的に評価を行い、学習済みスキルとして保存できる再利用可能なパターンを抽出します。

## 仕組み

このスキルは各セッション終了時に**Stopフック**として実行されます:

1. **セッション評価**: セッションが十分なメッセージ数（デフォルト: 10以上）を持っているか確認
2. **パターン検出**: セッションから抽出可能なパターンを特定
3. **スキル抽出**: 有用なパターンを `~/.claude/skills/learned/` に保存

## 設定

`config.json` を編集してカスタマイズできます:

```json
{
  "min_session_length": 10,
  "extraction_threshold": "medium",
  "auto_approve": false,
  "learned_skills_path": "~/.claude/skills/learned/",
  "patterns_to_detect": [
    "error_resolution",
    "user_corrections",
    "workarounds",
    "debugging_techniques",
    "project_specific"
  ],
  "ignore_patterns": [
    "simple_typos",
    "one_time_fixes",
    "external_api_issues"
  ]
}
```

## パターンの種類

| パターン | 説明 |
|---------|------|
| `error_resolution` | 特定のエラーがどのように解決されたか |
| `user_corrections` | ユーザーの修正から得られたパターン |
| `workarounds` | フレームワーク/ライブラリの癖に対する回避策 |
| `debugging_techniques` | 効果的なデバッグ手法 |
| `project_specific` | プロジェクト固有の規約 |

## フックの設定

`~/.claude/settings.json` に以下を追加してください:

```json
{
  "hooks": {
    "Stop": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/skills/continuous-learning/evaluate-session.sh"
      }]
    }]
  }
}
```

## なぜStopフックなのか？

- **軽量**: セッション終了時に一度だけ実行
- **非ブロッキング**: すべてのメッセージにレイテンシを追加しない
- **完全なコンテキスト**: セッション全体のトランスクリプトにアクセス可能

## 関連情報

- [ロングフォームガイド](https://x.com/affaanmustafa/status/2014040193557471352) - 継続学習に関するセクション
- `/learn` コマンド - セッション中の手動パターン抽出

---

## 比較ノート（調査: 2025年1月）

### Homunculusとの比較 (github.com/humanplane/homunculus)

Homunculus v2はより洗練されたアプローチを採用しています:

| 機能 | 我々のアプローチ | Homunculus v2 |
|------|-----------------|---------------|
| 観察 | Stopフック（セッション終了時） | PreToolUse/PostToolUseフック（100%信頼性） |
| 分析 | メインコンテキスト | バックグラウンドエージェント（Haiku） |
| 粒度 | 完全なスキル | アトミックな「インスティンクト」 |
| 信頼度 | なし | 0.3〜0.9の重み付き |
| 進化 | 直接スキルへ | インスティンクト → クラスター → スキル/コマンド/エージェント |
| 共有 | なし | インスティンクトのエクスポート/インポート |

**Homunculusからの重要な知見:**
> 「v1はスキルを使って観察していました。スキルは確率的で、50〜80%の確率で発動します。v2は観察にフック（100%信頼性）を使用し、学習された行動のアトミック単位としてインスティンクトを使用します。」

### 潜在的なv2の改善点

1. **インスティンクトベースの学習** - 信頼度スコアリング付きの小さなアトミックな行動
2. **バックグラウンドオブザーバー** - 並行して分析するHaikuエージェント
3. **信頼度の減衰** - 矛盾した場合にインスティンクトの信頼度が低下
4. **ドメインタグ付け** - code-style、testing、git、debuggingなど
5. **進化パス** - 関連するインスティンクトをスキル/コマンドにクラスター化

参照: `/Users/affoon/Documents/tasks/12-continuous-learning-v2.md` 完全な仕様書
