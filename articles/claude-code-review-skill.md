---
title: 'Claude Codeの/code-reviewスキルで自動コードレビューを仕組み化する'
emoji: '🔍'
type: 'tech'
topics:
  - 'ClaudeCode'
  - 'AI'
  - 'CodeReview'
  - 'CLI'
published: true
---

Claude Codeにはスキル（Custom Slash Commands）という仕組みがあります。`~/.claude/skills/`にMarkdownファイルを配置すると、`/スキル名`で呼び出せるカスタムコマンドになります。

この記事では、コード変更を自動レビューする`/code-review`スキルの構築方法を解説します。

## スキルとは

Claude Codeのスキルは、`~/.claude/skills/<スキル名>/SKILL.md`に配置するMarkdownファイルです。frontmatterでメタデータを定義し、本文にプロンプトを記述します。

```yaml
---
description: スキルの説明
context: fork # fork: 別コンテキストで実行、inherit: 現在のコンテキストを継承
agent: Explore # 使用するエージェントタイプ
---
```

`context: fork`を指定すると、現在の会話コンテキストとは独立したプロセスでスキルが実行されます。レビューのように「実装者とは別の視点でチェックしたい」用途に適しています。

## /code-reviewスキルの構成

```text
~/.claude/skills/code-review/
├── SKILL.md              ← メインのレビュープロンプト
└── languages/            ← 言語固有のチェックリスト
    ├── go.md
    ├── typescript.md
    ├── php.md
    ├── python.md
    └── rust.md
```

### SKILL.MDの設計ポイント

- **`context: fork`**: 実装中のコンテキストに引きずられず、独立した視点でレビューする
- **`agent: Explore`**: 読み取り専用のエージェントを使い、レビュー中にファイルを変更しない
- **レビュー対象はgit diff**: 変更差分だけを見るため、リポジトリ全体を読む必要がない

## 言語固有チェックリスト

`languages/`ディレクトリに言語ごとのチェックリストを配置すると、レビュー時に自動で参照されます。各ファイルには「よくあるミス」と「参考資料」をまとめます。

現在対応している言語はGo、TypeScript、PHP、Python、Rustです。自分のチームやプロジェクトで頻出するミスを追加していくと、レビュー精度が上がります。

## 使い方

### 対話モードで使う

Claude Codeのセッション中に`/code-review`と入力するだけです。

```text
> /code-review
```

`context: fork`により別プロセスで起動し、`git diff`の内容をレビューして結果を返します。実装中のコンテキストには影響しません。

### ヘッドレスモードで使う（claude-tmux連携）

claude-tmuxと組み合わせると、実装完了後に自動でレビューが走ります。

https://zenn.dev/because02/articles/claude-tmux-parallel-agents

claude-tmuxの`spawn`コマンドは、実装エージェント完了後にSKILL.MDを読み込み、2つ目の`claude -p`でレビューします。

```text
claude-tmux spawn "タスク"
  │
  ├── claude -p --allowedTools "Edit Write"  ← 実装
  ├── git diff → レビュープロンプト生成
  └── claude -p --allowedTools "Read"        ← レビュー（SKILL.MD使用）
```

実装エージェントにはEdit/Writeを許可し、レビューエージェントにはReadのみ許可しています。レビュー結果はログに出力されるだけで、ファイルを変更しません。

<!-- prettier-ignore -->
:::message alert
実装エージェントはEdit/Writeが承認なしで許可されています。意図しないファイル変更のリスクがあるため、完了後は`git diff`で変更内容を確認してください。
:::

## チェックリストの育て方

`/code-review`スキルの価値は、チェックリストを継続的に改善することで高まります。

### /update-review-checklistスキルとの連携

レビューで見逃したミスや新たに気づいた落とし穴は、`/update-review-checklist`スキルでチェックリストに追加できます。SKILL.MDの末尾にもこの導線を記載しています。

```markdown
**レビュー後のフィードバック:** このレビューで見逃したミスや新たに気づいた落とし穴があれば、/update-review-checklist スキルでチェックリストに追加してください。
```

### 追加の例

- プロジェクト固有のルール（命名規則、API設計方針）
- 過去に発生したバグのパターン
- 新しい言語の追加（`languages/`にファイルを追加するだけ）

## セットアップ

SKILL.MDと言語チェックリストを`~/.claude/skills/code-review/`に配置すれば動きます。実際のファイルはdotfilesリポジトリで公開しています。

https://github.com/a1yama/dotfiles/tree/master/packages/claude/.claude/skills/code-review

GNU Stowで管理している場合は`stow -d ~/dotfiles/packages -t ~ -R claude`で反映できます。

動作確認は、何かしらの変更がある状態でClaude Codeを開き`/code-review`と入力してください。

## まとめ

- `/code-review`はClaude Codeのスキル機能を使った自動コードレビューの仕組み
- `context: fork`と`agent: Explore`で実装コンテキストから独立した読み取り専用レビューを実現
- `languages/`に言語固有チェックリストを配置すると、レビュー観点が拡張される
- claude-tmux連携で実装→レビューの完全自動化が可能
- チェックリストを育てることでレビュー精度が継続的に向上する
