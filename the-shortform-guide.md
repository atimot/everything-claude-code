# Claude Code 完全ガイド（簡易版）

![Header: Anthropic Hackathon Winner - Tips & Tricks for Claude Code](./assets/images/shortform/00-header.png)

---

**2月の試験的リリース以来、熱心なClaude Codeユーザーとして使い続けてきました。[@DRodriguezFX](https://x.com/DRodriguezFX) と共に、Claude Codeのみを使用して Anthropic x Forum Ventures ハッカソンで [zenith.chat](https://zenith.chat) を開発し、優勝しました。**

10ヶ月間毎日使い続けた私の完全なセットアップを紹介します：スキル、フック、サブエージェント、MCP、プラグイン、そして実際に効果があったものについて。

---

## スキルとコマンド

スキルはルールのように機能し、特定のスコープやワークフローに限定されます。特定のワークフローを実行する際のプロンプトの省略形です。

Opus 4.5での長時間のコーディングセッション後に、デッドコードや不要な`.md`ファイルを整理したい場合は `/refactor-clean` を実行します。テストが必要な場合は `/tdd`、`/e2e`、`/test-coverage` を使います。スキルにはコードマップも含めることができます。コードマップは、コンテキストの探索に消費することなく、Claudeがコードベースを素早くナビゲートするための手段です。

![Terminal showing chained commands](./assets/images/shortform/02-chaining-commands.jpeg)
*コマンドのチェーン実行*

コマンドはスラッシュコマンドで実行されるスキルです。重複する部分もありますが、保存場所が異なります：

- **スキル**: `~/.claude/skills/` - より広範なワークフロー定義
- **コマンド**: `~/.claude/commands/` - すぐに実行できるプロンプト

```bash
# スキル構成の例
~/.claude/skills/
  pmx-guidelines.md      # プロジェクト固有のパターン
  coding-standards.md    # 言語のベストプラクティス
  tdd-workflow/          # README.mdを含む複数ファイルのスキル
  security-review/       # チェックリスト型スキル
```

---

## フック

フックは特定のイベントで発火するトリガーベースの自動化です。スキルとは異なり、ツール呼び出しとライフサイクルイベントに限定されます。

**フックの種類：**

1. **PreToolUse** - ツール実行前（バリデーション、リマインダー）
2. **PostToolUse** - ツール完了後（フォーマット、フィードバックループ）
3. **UserPromptSubmit** - メッセージ送信時
4. **Stop** - Claudeの応答完了時
5. **PreCompact** - コンテキスト圧縮前
6. **Notification** - パーミッション要求時

**例: 長時間実行コマンド前のtmuxリマインダー**

```json
{
  "PreToolUse": [
    {
      "matcher": "tool == \"Bash\" && tool_input.command matches \"(npm|pnpm|yarn|cargo|pytest)\"",
      "hooks": [
        {
          "type": "command",
          "command": "if [ -z \"$TMUX\" ]; then echo '[Hook] Consider tmux for session persistence' >&2; fi"
        }
      ]
    }
  ]
}
```

![PostToolUse hook feedback](./assets/images/shortform/03-posttooluse-hook.png)
*PostToolUseフック実行中のClaude Codeでのフィードバック表示例*

**プロのヒント:** `hookify` プラグインを使えば、JSONを手書きする代わりに会話形式でフックを作成できます。`/hookify` を実行して、やりたいことを説明するだけです。

---

## サブエージェント

サブエージェントは、オーケストレーター（メインのClaude）が限定されたスコープでタスクを委任できるプロセスです。バックグラウンドまたはフォアグラウンドで実行でき、メインエージェントのコンテキストを解放します。

サブエージェントはスキルとうまく連携します。スキルのサブセットを実行できるサブエージェントにタスクを委任すれば、それらのスキルを自律的に使用できます。また、特定のツールパーミッションでサンドボックス化することも可能です。

```bash
# サブエージェント構成の例
~/.claude/agents/
  planner.md           # 機能実装の計画
  architect.md         # システム設計の判断
  tdd-guide.md         # テスト駆動開発
  code-reviewer.md     # 品質・セキュリティレビュー
  security-reviewer.md # 脆弱性分析
  build-error-resolver.md
  e2e-runner.md
  refactor-cleaner.md
```

サブエージェントごとに許可するツール、MCP、パーミッションを設定し、適切なスコーピングを行いましょう。

---

## ルールとメモリ

`.rules` フォルダには、Claudeが**常に**従うべきベストプラクティスが記載された`.md`ファイルが格納されます。2つのアプローチがあります：

1. **単一のCLAUDE.md** - すべてを1つのファイルに（ユーザーレベルまたはプロジェクトレベル）
2. **ルールフォルダ** - 関心事ごとにグループ化されたモジュラーな`.md`ファイル

```bash
~/.claude/rules/
  security.md      # ハードコードされた秘密情報の禁止、入力のバリデーション
  coding-style.md  # イミュータビリティ、ファイル構成
  testing.md       # TDDワークフロー、80%カバレッジ
  git-workflow.md  # コミットフォーマット、PRプロセス
  agents.md        # サブエージェントへの委任タイミング
  performance.md   # モデル選択、コンテキスト管理
```

**ルールの例：**

- コードベースに絵文字を使用しない
- フロントエンドで紫系の色を避ける
- デプロイ前に必ずコードをテストする
- 巨大ファイルよりもモジュラーなコードを優先する
- console.logをコミットしない

---

## MCP（Model Context Protocol）

MCPはClaudeを外部サービスに直接接続します。APIの代替ではなく、APIのプロンプト駆動型ラッパーであり、情報のナビゲーションにおいてより柔軟性を提供します。

**例：** Supabase MCPを使えば、Claudeはコピー＆ペーストなしで特定のデータを取得したり、上流で直接SQLを実行できます。データベースやデプロイメントプラットフォームなども同様です。

![Supabase MCP listing tables](./assets/images/shortform/04-supabase-mcp.jpeg)
*Supabase MCPがpublicスキーマ内のテーブルを一覧表示している例*

**Chrome in Claude:** Claudeが自律的にブラウザを操作し、動作を確認できる組み込みプラグインMCPです。

**重要：コンテキストウィンドウの管理**

MCPの選定は慎重に。すべてのMCPをユーザー設定に入れていますが、**未使用のものはすべて無効化**しています。`/plugins` に移動してスクロールするか、`/mcp` を実行してください。

![/plugins interface](./assets/images/shortform/05-plugins-interface.jpeg)
*/pluginsを使用してMCPに移動し、現在インストールされているものとそのステータスを確認*

圧縮前の200kコンテキストウィンドウが、ツールを有効にしすぎると70kしか使えなくなることがあります。パフォーマンスが大幅に低下します。

**経験則：** 設定には20〜30のMCPを入れておき、有効にするのは10未満 / アクティブなツールは80未満に抑える。

```bash
# 有効なMCPの確認
/mcp

# ~/.claude.json の projects.disabledMcpServers で未使用のものを無効化
```

---

## プラグイン

プラグインは、面倒な手動セットアップの代わりに、ツールを簡単にインストールできるようにパッケージ化したものです。プラグインはスキル＋MCPの組み合わせや、フック/ツールをバンドルしたものになります。

**プラグインのインストール：**

```bash
# マーケットプレイスの追加
claude plugin marketplace add https://github.com/mixedbread-ai/mgrep

# Claudeを開き、/plugins を実行、新しいマーケットプレイスを見つけてインストール
```

![Marketplaces tab showing mgrep](./assets/images/shortform/06-marketplaces-mgrep.jpeg)
*新しくインストールされたMixedbread-Grepマーケットプレイスの表示*

**LSPプラグイン**は、エディタの外でClaude Codeを頻繁に使う場合に特に便利です。Language Server Protocolにより、IDEを開かずにリアルタイムの型チェック、定義へのジャンプ、インテリジェントな補完機能をClaudeに提供します。

```bash
# 有効なプラグインの例
typescript-lsp@claude-plugins-official  # TypeScriptインテリジェンス
pyright-lsp@claude-plugins-official     # Python型チェック
hookify@claude-plugins-official         # 会話形式でフック作成
mgrep@Mixedbread-Grep                   # ripgrepより優れた検索
```

MCPと同じ注意点 - コンテキストウィンドウに気をつけましょう。

---

## ヒントとコツ

### キーボードショートカット

- `Ctrl+U` - 行全体の削除（バックスペース連打より速い）
- `!` - クイックbashコマンドプレフィックス
- `@` - ファイル検索
- `/` - スラッシュコマンドの開始
- `Shift+Enter` - 複数行入力
- `Tab` - 思考プロセス表示の切り替え
- `Esc Esc` - Claudeの中断 / コードの復元

### 並列ワークフロー

- **フォーク** (`/fork`) - 重複しないタスクを並列で行うために会話をフォーク。メッセージのキューイングの代わりに使用
- **Gitワークツリー** - 重複する並列Claudeをコンフリクトなしで実行。各ワークツリーは独立したチェックアウト

```bash
git worktree add ../feature-branch feature-branch
# 各ワークツリーで別々のClaudeインスタンスを実行
```

### tmuxによる長時間実行コマンド

Claudeが実行するログやbashプロセスのストリーミングと監視：

https://github.com/user-attachments/assets/shortform/07-tmux-video.mp4

```bash
tmux new -s dev
# Claudeがここでコマンドを実行、デタッチしてリアタッチ可能
tmux attach -t dev
```

### mgrep > grep

`mgrep` はripgrep/grepから大幅に改善されたツールです。プラグインマーケットプレイスからインストールし、`/mgrep` スキルを使用します。ローカル検索とWeb検索の両方に対応。

```bash
mgrep "function handleSubmit"  # ローカル検索
mgrep --web "Next.js 15 app router changes"  # Web検索
```

### その他の便利なコマンド

- `/rewind` - 以前の状態に戻る
- `/statusline` - ブランチ、コンテキスト%、TODOでカスタマイズ
- `/checkpoints` - ファイルレベルの取り消しポイント
- `/compact` - コンテキスト圧縮の手動トリガー

### GitHub Actions CI/CD

GitHub Actionsを使ってPRのコードレビューを設定できます。設定すれば、ClaudeがPRを自動的にレビューします。

![Claude bot approving a PR](./assets/images/shortform/08-github-pr-review.jpeg)
*Claudeがバグ修正PRを承認*

### サンドボックス

リスクのある操作にはサンドボックスモードを使用 - Claudeは実際のシステムに影響を与えない制限された環境で実行されます。

---

## エディタについて

エディタの選択はClaude Codeのワークフローに大きく影響します。Claude Codeはどのターミナルからでも動作しますが、優れたエディタと組み合わせることで、リアルタイムのファイル追跡、素早いナビゲーション、統合されたコマンド実行が可能になります。

### Zed（私の好み）

私は[Zed](https://zed.dev)を使っています。Rustで書かれているので、本当に高速です。瞬時に開き、巨大なコードベースも問題なく処理し、システムリソースをほとんど消費しません。

**Zed + Claude Codeが優れた組み合わせである理由：**

- **速度** - Rustベースのパフォーマンスにより、Claudeがファイルを高速に編集してもラグなし。エディタがしっかり追従
- **エージェントパネル統合** - ZedのClaude統合により、Claudeの編集によるファイル変更をリアルタイムで追跡。エディタを離れずに参照ファイル間をジャンプ
- **CMD+Shift+R コマンドパレット** - カスタムスラッシュコマンド、デバッガー、ビルドスクリプトへの素早いアクセスを検索可能なUIで提供
- **最小限のリソース使用** - 重い操作中にClaudeとRAM/CPUを奪い合わない。Opus実行時に重要
- **Vimモード** - 完全なvimキーバインディングに対応

![Zed Editor with custom commands](./assets/images/shortform/09-zed-editor.jpeg)
*CMD+Shift+Rでカスタムコマンドドロップダウンを表示するZed Editor。右下にフォローモード（ブルズアイ）が表示*

**エディタに依存しないヒント：**

1. **画面を分割** - 片側にClaude Codeのターミナル、もう片側にエディタ
2. **Ctrl + G** - Claudeが現在作業しているファイルをZedで素早く開く
3. **自動保存** - 自動保存を有効にして、Claudeのファイル読み取りが常に最新になるようにする
4. **Git統合** - エディタのGit機能を使って、コミット前にClaudeの変更をレビュー
5. **ファイルウォッチャー** - ほとんどのエディタは変更されたファイルを自動リロード。これが有効になっていることを確認

### VSCode / Cursor

これも実用的な選択肢であり、Claude Codeとうまく連携します。`\ide` を使ってエディタと自動同期するターミナル形式で使用し、LSP機能を有効にするか（プラグインで冗長になった部分もあります）、エディタに統合されUIが一致するエクステンションを選択できます。

![VS Code Claude Code Extension](./assets/images/shortform/10-vscode-extension.jpeg)
*VS Codeエクステンションは、IDEに直接統合されたClaude Codeのネイティブグラフィカルインターフェースを提供*

---

## 私のセットアップ

### プラグイン

**インストール済み：** （通常、同時に有効にするのは4〜5個程度）

```markdown
ralph-wiggum@claude-code-plugins       # ループ自動化
frontend-design@claude-code-plugins    # UI/UXパターン
commit-commands@claude-code-plugins    # Gitワークフロー
security-guidance@claude-code-plugins  # セキュリティチェック
pr-review-toolkit@claude-code-plugins  # PR自動化
typescript-lsp@claude-plugins-official # TSインテリジェンス
hookify@claude-plugins-official        # フック作成
code-simplifier@claude-plugins-official
feature-dev@claude-code-plugins
explanatory-output-style@claude-code-plugins
code-review@claude-code-plugins
context7@claude-plugins-official       # ライブドキュメンテーション
pyright-lsp@claude-plugins-official    # Python型チェック
mgrep@Mixedbread-Grep                  # 優れた検索
```

### MCPサーバー

**設定済み（ユーザーレベル）：**

```json
{
  "github": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-github"] },
  "firecrawl": { "command": "npx", "args": ["-y", "firecrawl-mcp"] },
  "supabase": {
    "command": "npx",
    "args": ["-y", "@supabase/mcp-server-supabase@latest", "--project-ref=YOUR_REF"]
  },
  "memory": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-memory"] },
  "sequential-thinking": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-sequential-thinking"]
  },
  "vercel": { "type": "http", "url": "https://mcp.vercel.com" },
  "railway": { "command": "npx", "args": ["-y", "@railway/mcp-server"] },
  "cloudflare-docs": { "type": "http", "url": "https://docs.mcp.cloudflare.com/mcp" },
  "cloudflare-workers-bindings": {
    "type": "http",
    "url": "https://bindings.mcp.cloudflare.com/mcp"
  },
  "clickhouse": { "type": "http", "url": "https://mcp.clickhouse.cloud/mcp" },
  "AbletonMCP": { "command": "uvx", "args": ["ableton-mcp"] },
  "magic": { "command": "npx", "args": ["-y", "@magicuidesign/mcp@latest"] }
}
```

ここが重要なポイント - 14のMCPを設定していますが、プロジェクトごとに有効にするのは5〜6個程度。コンテキストウィンドウを健全に保ちます。

### 主要フック

```json
{
  "PreToolUse": [
    { "matcher": "npm|pnpm|yarn|cargo|pytest", "hooks": ["tmuxリマインダー"] },
    { "matcher": "Write && .md file", "hooks": ["README/CLAUDE以外はブロック"] },
    { "matcher": "git push", "hooks": ["レビュー用にエディタを開く"] }
  ],
  "PostToolUse": [
    { "matcher": "Edit && .ts/.tsx/.js/.jsx", "hooks": ["prettier --write"] },
    { "matcher": "Edit && .ts/.tsx", "hooks": ["tsc --noEmit"] },
    { "matcher": "Edit", "hooks": ["grep console.log 警告"] }
  ],
  "Stop": [
    { "matcher": "*", "hooks": ["変更ファイルのconsole.logチェック"] }
  ]
}
```

### カスタムステータスライン

ユーザー名、ディレクトリ、ダーティインジケーター付きgitブランチ、コンテキスト残量%、モデル、時刻、TODO数を表示：

![Custom status line](./assets/images/shortform/11-statusline.jpeg)
*Macルートディレクトリでのステータスライン例*

```
affoon:~ ctx:65% Opus 4.5 19:52
▌▌ plan mode on (shift+tab to cycle)
```

### ルール構成

```
~/.claude/rules/
  security.md      # 必須のセキュリティチェック
  coding-style.md  # イミュータビリティ、ファイルサイズ制限
  testing.md       # TDD、80%カバレッジ
  git-workflow.md  # コンベンショナルコミット
  agents.md        # サブエージェント委任ルール
  patterns.md      # APIレスポンスフォーマット
  performance.md   # モデル選択（Haiku vs Sonnet vs Opus）
  hooks.md         # フックのドキュメント
```

### サブエージェント

```
~/.claude/agents/
  planner.md           # 機能の分解
  architect.md         # システム設計
  tdd-guide.md         # テストファースト
  code-reviewer.md     # 品質レビュー
  security-reviewer.md # 脆弱性スキャン
  build-error-resolver.md
  e2e-runner.md        # Playwrightテスト
  refactor-cleaner.md  # デッドコード削除
  doc-updater.md       # ドキュメント同期
```

---

## 重要なポイント

1. **複雑にしすぎない** - 設定はアーキテクチャではなく、ファインチューニングとして扱う
2. **コンテキストウィンドウは貴重** - 未使用のMCPとプラグインを無効にする
3. **並列実行** - 会話のフォーク、gitワークツリーを活用
4. **繰り返し作業の自動化** - フォーマット、リンティング、リマインダーにフックを使用
5. **サブエージェントのスコープを絞る** - ツールを限定 = 集中した実行

---

## リファレンス

- [プラグインリファレンス](https://code.claude.com/docs/en/plugins-reference)
- [フックドキュメント](https://code.claude.com/docs/en/hooks)
- [チェックポイント](https://code.claude.com/docs/en/checkpointing)
- [インタラクティブモード](https://code.claude.com/docs/en/interactive-mode)
- [メモリシステム](https://code.claude.com/docs/en/memory)
- [サブエージェント](https://code.claude.com/docs/en/sub-agents)
- [MCP概要](https://code.claude.com/docs/en/mcp-overview)

---

**注：** これは詳細の一部です。高度なパターンについては[詳細版ガイド](./the-longform-guide.md)を参照してください。

---

*NYCで開催された Anthropic x Forum Ventures ハッカソンにて、[@DRodriguezFX](https://x.com/DRodriguezFX) と共に [zenith.chat](https://zenith.chat) を開発し優勝*
