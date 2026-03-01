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

### SKILL.MD

```markdown
---
description: コード変更のレビューを独立したエージェントで実行する
context: fork
agent: Explore
---

## レビュー対象

以下のコマンドで変更差分を取得し、レビューしてください。

git diff git diff --cached

未追跡の新規ファイルがある場合は、ファイル内容を直接読んでレビューしてください。

git status --short

## レビュー観点

1. **バグ・ロジックエラー** — 意図しない動作、エッジケースの見落とし、off-by-oneエラー
2. **セキュリティ** — インジェクション、ハードコードされた秘密情報、安全でない入力処理
3. **可読性・保守性** — 不明瞭な命名、不必要な複雑さ、過剰な抽象化
4. **一貫性** — 既存コードベースのスタイル・パターンとの整合性

## 言語固有のチェックポイント

変更されたファイルの言語に応じて、languages/ディレクトリ内の該当ファイルを参照してください。

## 出力フォーマット

問題がある場合のみ報告してください。問題がなければ「問題なし」と報告してください。

各指摘は以下の形式で簡潔に:

[重要度: high/medium/low] ファイルパス:行番号内容: 問題の説明提案: 修正案
```

ポイントは3つです。

- **`context: fork`**: 実装中のコンテキストに引きずられず、独立した視点でレビューする
- **`agent: Explore`**: 読み取り専用のエージェントを使い、レビュー中にファイルを変更しない
- **レビュー対象はgit diff**: 変更差分だけを見るため、リポジトリ全体を読む必要がない

## 言語固有チェックリスト

`languages/`ディレクトリに言語ごとのチェックリストを配置すると、レビュー時に自動で参照されます。各ファイルには「よくあるミス」と「参考資料」をまとめます。

### Goの例（languages/go.md）

```markdown
# Go レビューチェックリスト

## よくあるミス

### 定数定義

- **iotaを使ったconstの途中に新しい定数を追加**
  - 説明: iotaで定義された定数の途中に追加すると、以降の値が全てずれる
  - 対策: 新しい定数は末尾に追加するか、明示的に値を指定する

### エラーハンドリング

- **エラーの無視（\_ = err）**
- **エラーのラップ不足（%wを使わない）**

### 並行処理

- **goroutineリーク（contextキャンセルの欠如）**
- **deferのループ内使用**

## 参考資料

- [Effective Go](https://go.dev/doc/effective_go)
- [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)
```

### TypeScript/JavaScriptの例（languages/typescript.md）

```markdown
# TypeScript/JavaScript レビューチェックリスト

## よくあるミス

### 型安全性

- **anyの多用（unknownや具体的な型を使うべき）**
- **型アサーション（as）の乱用（型ガードを使うべき）**

### 非同期処理

- **PromiseのUnhandled Rejection**
- **並列実行可能な処理をawaitで直列実行**

### React固有

- **useEffectの依存配列の不備**
- **key propの欠如**

## 参考資料

- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- [Google TypeScript Style Guide](https://google.github.io/styleguide/tsguide.html)
```

他の言語（PHP、Python、Rust）も同じ構造で作成できます。自分のチームやプロジェクトで頻出するミスを追加していくと、レビュー精度が上がります。

## 使い方

### 対話モードで使う

Claude Codeのセッション中に`/code-review`と入力するだけです。

```text
> /code-review
```

`context: fork`により別プロセスで起動し、`git diff`の内容をレビューして結果を返します。実装中のコンテキストには影響しません。

### ヘッドレスモードで使う（claude-tmux連携）

[claude-tmux](https://zenn.dev/because02/articles/claude-tmux-parallel-agents)と組み合わせると、実装完了後に自動でレビューが走ります。

claude-tmuxの`spawn`コマンドは、実装エージェント完了後にSKILL.MDを読み込み、2つ目の`claude -p`でレビューします。

```text
claude-tmux spawn "タスク"
  │
  ├── claude -p --allowedTools "Edit Write"  ← 実装
  ├── git diff → レビュープロンプト生成
  └── claude -p --allowedTools "Read"        ← レビュー（SKILL.MD使用）
```

実装エージェントにはEdit/Writeを許可し、レビューエージェントにはReadのみ許可しています。レビュー結果はログに出力されるだけで、ファイルを変更しません。

:::message alert実装エージェントはEdit/Writeが承認なしで許可されています。意図しないファイル変更のリスクがあるため、完了後は`git diff`で変更内容を確認してください。:::

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

## セットアップ手順

### 1. ディレクトリ作成

```bash
mkdir -p ~/.claude/skills/code-review/languages
```

### 2. SKILL.MDを配置

上記のSKILL.MDの内容を`~/.claude/skills/code-review/SKILL.md`に保存します。

### 3. 言語チェックリストを配置

使用する言語のチェックリストを`languages/`に配置します。不要な言語はスキップしても問題ありません。

### 4. 動作確認

```bash
# 何かしらの変更を加えた状態で
claude
> /code-review
```

### dotfilesで管理する場合

実際のファイルは[dotfilesリポジトリ](https://github.com/a1yama/dotfiles/tree/master/packages/claude/.claude/skills/code-review)で公開しています。

GNU Stowでdotfilesを管理している場合は、以下のパスに配置します。

```text
dotfiles/packages/claude/.claude/skills/code-review/SKILL.md
dotfiles/packages/claude/.claude/skills/code-review/languages/go.md
dotfiles/packages/claude/.claude/skills/code-review/languages/typescript.md
```

```bash
stow -d ~/dotfiles/packages -t ~ -R claude
```

## まとめ

- `/code-review`はClaude Codeのスキル機能を使った自動コードレビューの仕組み
- `context: fork`と`agent: Explore`で実装コンテキストから独立した読み取り専用レビューを実現
- `languages/`に言語固有チェックリストを配置すると、レビュー観点が拡張される
- claude-tmux連携で実装→レビューの完全自動化が可能
- チェックリストを育てることでレビュー精度が継続的に向上する
