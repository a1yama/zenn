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

```text
dotfiles/packages/claude/.local/bin/claude-tmux
```

```bash
stow -d ~/dotfiles/packages -t ~ -R claude
```

## コマンド一覧

### spawn：エージェントの起動

```bash
claude-tmux spawn "タスクの説明" --name エージェント名 --dir 作業ディレクトリ
```

tmux内に新しいウィンドウ`🤖エージェント名`が作成され、ヘッドレスエージェントが起動します。

```bash
# 基本的な使い方
claude-tmux spawn "GET /health エンドポイントを追加" --name health

# 作業ディレクトリを指定
claude-tmux spawn "READMEを更新" --name docs --dir ~/projects/my-app

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

:::message alertエージェントはファイルの作成・編集を確認なしで実行します。意図しないファイル変更のリスクがあるため、完了後は必ず`git diff`で変更内容を確認してください。:::

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

:::message Bashを広く許可すると任意コマンド実行のリスクがあります。パターンで必ず絞ってください。:::

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
4. 完了時に`say`で音声通知、`read`で待機

```text
claude-tmux spawn "タスク"
  │
  ├── /tmp/claude-agents/name.prompt    ← プロンプト
  ├── /tmp/claude-agents/name.runner.sh ← 実行スクリプト
  ├── /tmp/claude-agents/name.log       ← 実行ログ
  │
  └── tmux new-window → runner.sh → claude -p → 完了 → say → read
```

ウィンドウ名に`🤖`プレフィックスを付けることで、`status`コマンドがエージェントウィンドウだけをフィルタリングできます。

## ソースコード全文

約250行のbashスクリプトです。特別な依存はなく、tmuxとClaude Codeだけで動きます。

:::details claude-tmux全文

```bash
#!/usr/bin/env bash
set -euo pipefail

AGENT_PREFIX="🤖"
LOG_DIR="/tmp/claude-agents"

usage() {
  cat <<'EOF'
claude-tmux - tmux × Claude Code 並列エージェント管理

Usage:
  claude-tmux spawn "タスク説明"  [--name NAME] [--dir DIR]
  claude-tmux issues 42 43 44
  claude-tmux status
  claude-tmux logs <name>
  claude-tmux kill [name|all]

Subcommands:
  spawn   新しいtmuxウィンドウでヘッドレスエージェントを起動
  issues  複数のGitHub Issueを並列エージェントで処理
  status  実行中のエージェントウィンドウ一覧と最新ログを表示
  logs    特定エージェントのログを less で表示
  kill    エージェントウィンドウを停止

Note:
  spawn/issues で起動されるエージェントは Edit/Write が承認なしで許可されています。
  エージェントはファイルの作成・編集を確認なしで実行します。
  意図しないファイル変更のリスクがあるため、実行後に git diff で変更内容を確認してください。
EOF
}

require_tmux() {
  if [[ -z "${TMUX:-}" ]]; then
    echo "Error: tmuxセッション内で実行してください" >&2
    exit 1
  fi
}

ensure_log_dir() {
  mkdir -p "$LOG_DIR"
}

cmd_spawn() {
  require_tmux
  ensure_log_dir

  local name=""
  local dir=""
  local prompt=""

  while [[ $# -gt 0 ]]; do
    case "$1" in
      --name)
        name="$2"
        shift 2
        ;;
      --dir)
        dir="$2"
        shift 2
        ;;
      *)
        if [[ -z "$prompt" ]]; then
          prompt="$1"
        else
          prompt="$prompt $1"
        fi
        shift
        ;;
    esac
  done

  if [[ -z "$prompt" ]]; then
    echo "Error: タスク説明を指定してください" >&2
    echo "Usage: claude-tmux spawn \"タスク説明\" [--name NAME] [--dir DIR]" >&2
    exit 1
  fi

  if [[ -z "$name" ]]; then
    name="agent-$(date +%s)"
  fi

  local window_name="${AGENT_PREFIX}${name}"
  local log_file="${LOG_DIR}/${name}.log"
  local work_dir="${dir:-$(pwd)}"

  local prompt_file="${LOG_DIR}/${name}.prompt"
  printf '%s' "$prompt" > "$prompt_file"

  local runner="${LOG_DIR}/${name}.runner.sh"
  cat > "$runner" <<RUNNER
#!/usr/bin/env bash
name="${name}"
log_file="${log_file}"
prompt_file="${prompt_file}"

echo "=== Claude Agent: \${name} ===" | tee "\${log_file}"
echo "Prompt: \$(cat "\${prompt_file}")" | tee -a "\${log_file}"
echo "Dir: \$(pwd)" | tee -a "\${log_file}"
echo "---" | tee -a "\${log_file}"

# NOTE: Edit/Write は承認なしで許可されている（--allowedTools）。
# Bash等の他ツールは許可されていないため、ヘッドレスモードではスキップされる。
claude -p --allowedTools "Edit Write" "\$(cat "\${prompt_file}")" 2>&1 | tee -a "\${log_file}"

echo ""
echo "=== 完了 ==="
say "Claude agent \${name} が完了しました"
rm -f "\${prompt_file}" "\${runner:-}"
read -r -p "Enterで閉じる..."
RUNNER
  chmod +x "$runner"

  tmux new-window -n "$window_name" -c "$work_dir" "$runner"

  echo "Agent '${name}' を起動しました (window: ${window_name})"
  echo "Log: ${log_file}"
}

cmd_issues() {
  require_tmux
  ensure_log_dir

  if [[ $# -eq 0 ]]; then
    echo "Error: Issue番号を指定してください" >&2
    echo "Usage: claude-tmux issues 42 43 44" >&2
    exit 1
  fi

  for issue_num in "$@"; do
    local prompt="GitHub Issue #${issue_num} を \`gh issue view ${issue_num}\` で確認し、内容に基づいてタスクを遂行してください。"
    cmd_spawn "$prompt" --name "issue-${issue_num}"
  done
}

cmd_status() {
  require_tmux

  local windows
  windows=$(tmux list-windows -F '#{window_name}' 2>/dev/null | grep "^${AGENT_PREFIX}" || true)

  if [[ -z "$windows" ]]; then
    echo "実行中のエージェントはありません"
    return
  fi

  echo "=== 実行中のエージェント ==="
  echo ""

  while IFS= read -r window_name; do
    local name="${window_name#${AGENT_PREFIX}}"
    local log_file="${LOG_DIR}/${name}.log"

    echo "  ${window_name}"
    if [[ -f "$log_file" ]]; then
      echo "    Log: ${log_file}"
      echo "    最新:"
      tail -3 "$log_file" 2>/dev/null | sed 's/^/      /'
    fi
    echo ""
  done <<< "$windows"
}

cmd_logs() {
  if [[ $# -eq 0 ]]; then
    echo "Error: エージェント名を指定してください" >&2
    echo "Usage: claude-tmux logs <name>" >&2
    exit 1
  fi

  local name="$1"
  local log_file="${LOG_DIR}/${name}.log"

  if [[ ! -f "$log_file" ]]; then
    echo "Error: ログファイルが見つかりません: ${log_file}" >&2
    exit 1
  fi

  less +F "$log_file"
}

cmd_kill() {
  require_tmux

  if [[ $# -eq 0 ]]; then
    echo "Error: エージェント名または 'all' を指定してください" >&2
    echo "Usage: claude-tmux kill [name|all]" >&2
    exit 1
  fi

  local target="$1"

  if [[ "$target" == "all" ]]; then
    local windows
    windows=$(tmux list-windows -F '#{window_name}' 2>/dev/null | grep "^${AGENT_PREFIX}" || true)

    if [[ -z "$windows" ]]; then
      echo "実行中のエージェントはありません"
      return
    fi

    while IFS= read -r window_name; do
      tmux kill-window -t ":${window_name}" 2>/dev/null || true
      echo "停止: ${window_name}"
    done <<< "$windows"
  else
    local window_name="${AGENT_PREFIX}${target}"
    if tmux kill-window -t ":${window_name}" 2>/dev/null; then
      echo "停止: ${window_name}"
    else
      echo "Error: エージェント '${target}' が見つかりません" >&2
      exit 1
    fi
  fi
}

case "${1:-}" in
  spawn)  shift; cmd_spawn "$@" ;;
  issues) shift; cmd_issues "$@" ;;
  status) cmd_status ;;
  logs)   shift; cmd_logs "$@" ;;
  kill)   shift; cmd_kill "$@" ;;
  -h|--help|"") usage ;;
  *)
    echo "Error: 不明なサブコマンド '$1'" >&2
    usage
    exit 1
    ;;
esac
```

:::

## まとめ

- Claude Codeの`claude -p`とtmuxを組み合わせるだけで並列エージェント管理ができる
- claude-tmuxは約250行のbashスクリプトで、特別な依存はない
- `spawn`でタスク単位、`issues`でGitHub Issue単位の並列処理が可能
- Edit/Writeは`--allowedTools`でセッション単位の許可とし、完了後の`git diff`確認は必須
- 個人の日常開発ならこれで十分
