---
title: 'claude-tmux 実践ガイド — Claude Codeの並列エージェントをtmuxで管理する'
emoji: '🤖'
type: 'tech'
topics:
  - 'ClaudeCode'
  - 'tmux'
  - 'AI'
  - 'CLI'
published: true
---

Claude Codeは1セッション1タスクが基本です。複数タスクを並行して進めたくても、ターミナルを何個も開いて手動管理するのは現実的と言えません。

この記事では、tmuxのウィンドウ管理を活用してClaude Codeのヘッドレスエージェントを並列実行するCLIツール「claude-tmux」の導入から実践的な使い方までを解説します。

## この記事で得られるもの

- Claude Codeのヘッドレスモード（`claude -p`）の活用方法
- tmuxウィンドウを使った並列エージェント管理の仕組み
- claude-tmuxのインストール・設定・全コマンド解説
- パーミッション設定と放置運用のポイント
- 実装完了後の自動コードレビュー
- 実際の並列開発ワークフロー例

## 対象読者

- Claude Codeを日常的に使っているエンジニア
- 複数タスクを並行して処理したい方
- tmuxを普段から使っている方（必須）

## 前提環境

- macOSまたはLinux
- Claude Codeがインストール済み
- tmuxを使用中であること

## 課題：Claude Codeの1セッション制約

Claude Codeは優秀ですが、1つのセッションで1つのタスクしか処理できません。たとえばこんな状況を考えてみてください。

- Issue #42（認証機能の追加）を実装中
- Issue #43（バリデーション修正）も急ぎで対応が必要
- Issue #44（テスト追加）は放置しておきたい

Claude Codeへ順番にお願いすると、3タスク分の待ち時間が直列で発生します。並列化できるならそうしたいところです。

## 解決策：tmux + ヘッドレスモード

Claude Codeには`claude -p`というヘッドレスモード（非対話モード）があります。プロンプトを渡すと、対話なしでタスクを実行して終了します。

```bash
claude -p "GET /health エンドポイントを追加してください"
```

これをtmuxのウィンドウごとに起動すれば、並列実行が実現できます。claude-tmuxはこの仕組みをラップしたCLIツールです。

## インストール

claude-tmuxは単一のbashスクリプトです。`~/.local/bin/`に配置してPATHを通せば使えます。

```bash
# ファイルを配置
mkdir -p ~/.local/bin
# claude-tmuxスクリプトを ~/.local/bin/claude-tmux に作成
chmod +x ~/.local/bin/claude-tmux

# PATHに ~/.local/bin が含まれていない場合
export PATH="$HOME/.local/bin:$PATH"
```

dotfilesで管理する場合は、GNU Stowでシンボリックリンクを展開するのがおすすめです。

https://github.com/a1yama/dotfiles/tree/master/packages/claude/.local/bin

```bash
stow -d ~/dotfiles/packages -t ~ -R claude
```

## コマンド一覧

### spawn：エージェントの起動

```bash
claude-tmux spawn "タスクの説明" --name エージェント名 --dir 作業ディレクトリ
```

tmux内に新しいウィンドウ`🤖エージェント名`が作成され、ヘッドレスエージェントが起動します。実装が完了すると自動でコードレビューも実行されます。

```bash
# 基本的な使い方（実装後に自動レビュー）
claude-tmux spawn "GET /health エンドポイントを追加" --name health

# 作業ディレクトリを指定
claude-tmux spawn "READMEを更新" --name docs --dir ~/projects/my-app

# レビューをスキップ
claude-tmux spawn "テストを追加" --no-review

# --name を省略すると自動生成される（agent-1709312345 のような名前）
claude-tmux spawn "テストを追加"
```

`--name`は後続のコマンド（logs、kill）で使うので、複数エージェントを動かすなら付けておくと便利です。

### issues：GitHub Issueの並列処理

```bash
claude-tmux issues 42 43 44
```

Issue番号を列挙すると、各Issueに対して`gh issue view`で内容を確認し、タスクを遂行するエージェントが並列で起動します。内部的には`spawn`を繰り返し呼んでいるだけです。

### status：実行状況の確認

```bash
claude-tmux status
```

```text
=== 実行中のエージェント ===

  🤖health
    Log: /tmp/claude-agents/health.log
    最新:
      ファイルを作成しました: src/routes/health.ts
      テストを実行中...
      All tests passed.

  🤖auth
    Log: /tmp/claude-agents/auth.log
    最新:
      認証ミドルウェアを実装中...
```

各エージェントのログ末尾3行が表示されます。

### logs：ログの詳細確認

```bash
claude-tmux logs health
```

`less +F`で開くので、リアルタイムにログを追跡できます。`Ctrl+C`で追跡を止めて通常のlessモードに切り替え、`F`で追跡に戻れます。

### kill：エージェントの停止

```bash
# 特定のエージェントを停止
claude-tmux kill health

# 全エージェントを一括停止
claude-tmux kill all
```

## パーミッションと放置運用

### ヘッドレスモードのパーミッション問題

`claude -p`では対話的な承認プロンプトを出せません。許可されていないツールの呼び出しはスキップされます。つまり、Edit/Writeが未許可だと次のような問題が起きます。

1. エージェントがファイルを編集しようとする
2. スキップされる
3. 変更されていないので再度試みる
4. **無限ループ（トークンだけ消費して成果物ゼロ）**

これを防ぐため、claude-tmuxでは`--allowedTools`オプションでEdit/Writeをセッション単位で許可しています。

```bash
# claude-tmux内部の実行コマンド
claude -p --allowedTools "Edit Write" "タスクの説明"
```

この許可はセッション単位なので、`~/.claude/settings.json`には影響しません。通常のClaude Codeセッションのパーミッションはそのまま維持されます。

### 注意事項

<!-- prettier-ignore -->
:::message alert
エージェントはファイルの作成・編集を確認なしで実行します。意図しないファイル変更のリスクがあるため、完了後は必ず`git diff`で変更内容を確認してください。
:::

```bash
# エージェント完了後の確認フロー
git diff                    # 変更内容を確認
git diff --stat             # 変更ファイル一覧
git checkout -- .           # 問題があれば全変更を破棄
```

### Bashコマンドについて

現在の設定ではBashは許可していません。つまり`npm test`や`go build`などのコマンドはスキップされます。テスト実行まで含めて自動化したい場合は、`--allowedTools`にBashパターンを追加します。

```bash
# claude-tmuxのスクリプトを編集して追加
claude -p --allowedTools "Edit Write Bash(npm:*) Bash(go:*)" "タスクの説明"
```

<!-- prettier-ignore -->
:::message
Bashを広く許可すると任意コマンド実行のリスクがあります。パターンで必ず絞ってください。
:::

## 自動コードレビュー

claude-tmuxは実装完了後に`/code-review`スキル（`~/.claude/skills/code-review/SKILL.md`）を使って自動でコードレビューします。実装エージェントとは別の`claude -p`プロセスがレビューを担当するため、独立した視点でチェックが走ります。

https://zenn.dev/because02/articles/claude-code-review-skill

### フロー

1. 実装エージェントが`claude -p --allowedTools "Edit Write"`でタスクを完了
2. `git status --short`で変更の有無を確認
3. `~/.claude/skills/code-review/SKILL.md`からレビュープロンプトを読み込む
4. `git diff`の出力と組み合わせて2つ目の`claude -p --allowedTools "Read"`に渡す
5. レビュー結果がログに出力される

レビューエージェントにはReadのみ許可しています。言語固有のチェックリスト（`languages/*.md`）を参照できますが、ファイルの変更はしません。

### レビュー観点（SKILL.MDで定義）

SKILL.MDに定義されたレビュー観点が使用されます。

- バグ・ロジックエラー — 意図しない動作、エッジケースの見落とし
- セキュリティ — インジェクション、ハードコードされた秘密情報
- 可読性・保守性 — 不明瞭な命名、不必要な複雑さ
- 一貫性 — 既存コードベースのスタイルとの整合性

言語固有のチェックポイント（Go、TypeScript、PHP、Python、Rust）も`languages/`ディレクトリにあれば参照されます。レビュー結果は`[重要度: high/medium/low] ファイルパス:行番号`の形式でログに出力されます。問題がなければ「問題なし」と報告されます。

### スキルのカスタマイズ

レビュー観点を変更したい場合は`~/.claude/skills/code-review/SKILL.md`を編集してください。claude-tmux自体の修正は不要です。

### レビューのスキップ

READMEの更新や調査タスクなど、レビューが不要な場合は`--no-review`で無効にできます。

```bash
claude-tmux spawn "READMEを更新" --name docs --no-review
```

<!-- prettier-ignore -->
:::message alert
実装エージェントはEdit/Writeが承認なしで許可されています。レビューエージェントは読み取り専用です。レビュー結果に問題が報告された場合、修正は手動で行うか、新たにspawnしてください。
:::

## 実践ワークフロー

### パターン1：複数Issueの並列処理

```bash
# 3つのIssueを並列に処理
claude-tmux issues 42 43 44

# 状況を確認
claude-tmux status

# 完了通知（say）を待つ

# 変更を確認
git diff
```

### パターン2：大きなタスクを分解して並列処理

```bash
# 認証機能を3つのサブタスクに分解
claude-tmux spawn "JWTトークンの発行・検証ユーティリティを作成" --name jwt
claude-tmux spawn "認証ミドルウェアを作成" --name middleware
claude-tmux spawn "認証ルート（POST /login, POST /register）を追加" --name routes
```

ただし、タスク間に依存関係がある場合は注意が必要です。上の例ではmiddlewareがjwtに依存するので、厳密には順番に実行すべきかもしれません。**依存関係がないタスクだけを並列にするのが安全です。**

### パターン3：調査と実装の同時進行

```bash
# 調査エージェント
claude-tmux spawn "src/auth/ のコードを読んで、認証フローの全体像をまとめてREPORT.mdに書き出して" --name research

# 別のリポジトリで実装エージェント
claude-tmux spawn "ヘルスチェックAPIを追加" --name impl --dir ~/projects/other-app
```

## 仕組みの解説

claude-tmuxの内部動作はシンプルです。

1. プロンプトをファイルに書き出す（シェルのクォーティング問題を回避）
2. ラッパースクリプトを`/tmp/claude-agents/`に生成
3. `tmux new-window`でスクリプトを実行
4. 実装完了後、`git diff`を収集してレビュー用`claude -p`を実行
5. 完了時に`say`で音声通知、`read`で待機

```text
claude-tmux spawn "タスク"
  │
  ├── /tmp/claude-agents/name.prompt         ← プロンプト
  ├── /tmp/claude-agents/name.runner.sh      ← 実行スクリプト
  ├── /tmp/claude-agents/name.log            ← 実行ログ
  ├── /tmp/claude-agents/name.review-prompt  ← レビュープロンプト（一時）
  │
  └── tmux new-window → runner.sh
        ├── claude -p (実装) → 完了
        ├── git diff → レビュープロンプト生成
        ├── claude -p (レビュー) → 完了
        └── say → read
```

ウィンドウ名に`🤖`プレフィックスを付けることで、`status`コマンドがエージェントウィンドウだけをフィルタリングできます。

## ソースコード

約300行のbashスクリプトです。特別な依存はなく、tmuxとClaude Codeだけで動きます。

https://github.com/a1yama/dotfiles/blob/master/packages/claude/.local/bin/claude-tmux

## まとめ

- Claude Codeの`claude -p`とtmuxを組み合わせるだけで並列エージェント管理ができる
- claude-tmuxは約300行のbashスクリプトで、特別な依存はない
- `spawn`でタスク単位、`issues`でGitHub Issue単位の並列処理が可能
- 実装完了後に自動でコードレビューが走り、問題があればログに報告される
- 実装エージェントのEdit/Writeは`--allowedTools`でセッション単位の許可。レビューエージェントは読み取り専用
- 完了後の`git diff`確認は必須。個人の日常開発ならこれで十分
