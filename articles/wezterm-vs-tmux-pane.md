---
title: 'WezTermのpane機能で満足していたけど、結局tmuxに移行した話'
emoji: '🖥️'
type: 'tech'
topics:
  - 'WezTerm'
  - 'tmux'
  - 'ターミナル'
  - 'CLI'
published: false
---

## はじめに

ターミナルを分割して作業したいとき、いくつかの選択肢があります。ターミナルエミュレータ自体の機能を使う方法（WezTerm、iTerm2、Kittyなど）や、tmux・Zellij・screenといったターミナルマルチプレクサを使う方法などです。

私は普段WezTermを使っていて、paneとタブの操作で十分事足りていました。tmuxは多機能だけど覚えることが多そうだし、WezTermで困っていないならわざわざ使わなくてもいいかな、と思っていました。

ただ、ふと考えました。**WezTermをやめたらどうなる？**

WezTermのpane操作に慣れてしまうと、別のターミナルエミュレータに移行したときに同じ操作ができません。一方、tmuxを使っていれば、どのターミナルエミュレータでも同じ操作感で作業できます。

この記事では、WezTermとtmuxそれぞれのpane操作を紹介し、最終的にtmuxに移行して工夫している点をまとめます。

## WezTermのpane/タブ操作

WezTermはデフォルトで直感的なキーバインドが設定されています。

### pane操作

| 操作 | macOS | Windows/Linux |
|------|-------|---------------|
| 右に分割 | `Cmd + Shift + Alt + %` | `Ctrl + Shift + Alt + %` |
| 下に分割 | `Cmd + Shift + Alt + "` | `Ctrl + Shift + Alt + "` |
| pane間の移動 | `Ctrl + Shift + Arrow` | `Ctrl + Shift + Arrow` |
| paneを閉じる | `Cmd + w` | `Ctrl + Shift + w` |

### タブ操作

| 操作 | macOS | Windows/Linux |
|------|-------|---------------|
| 新しいタブ | `Cmd + t` | `Ctrl + Shift + t` |
| タブを閉じる | `Cmd + w` | `Ctrl + Shift + w` |
| 次のタブ | `Ctrl + Tab` | `Ctrl + Tab` |
| 前のタブ | `Ctrl + Shift + Tab` | `Ctrl + Shift + Tab` |
| タブ番号で移動 | `Cmd + 数字` | `Alt + 数字` |

macOSであれば`Cmd + t`でタブ作成、`Cmd + w`で閉じるなど、一般的なアプリケーションと同じ操作感で使えます。

## tmuxのpane/ウィンドウ操作

tmuxでは、操作の前にプレフィックスキー（デフォルトは `Ctrl + b`）を押す必要があります。

### pane操作

| 操作 | キーバインド |
|------|-------------|
| 左右に分割 | `Prefix` → `%` |
| 上下に分割 | `Prefix` → `"` |
| pane間の移動 | `Prefix` → `Arrow` |
| paneを閉じる | `Prefix` → `x` |
| paneのサイズ変更 | `Prefix` → `Ctrl + Arrow` |
| paneの入れ替え | `Prefix` → `{` / `}` |
| paneの最大化/復元 | `Prefix` → `z` |

### ウィンドウ（タブ相当）操作

| 操作 | キーバインド |
|------|-------------|
| 新しいウィンドウ | `Prefix` → `c` |
| ウィンドウを閉じる | `Prefix` → `&` |
| 次のウィンドウ | `Prefix` → `n` |
| 前のウィンドウ | `Prefix` → `p` |
| ウィンドウ番号で移動 | `Prefix` → `数字` |
| ウィンドウ一覧 | `Prefix` → `w` |

### セッション操作

| 操作 | キーバインド |
|------|-------------|
| セッションをデタッチ | `Prefix` → `d` |
| セッション一覧 | `Prefix` → `s` |
| セッションにアタッチ | `tmux attach -t <name>` |
| 新しいセッション | `tmux new -s <name>` |

## tmuxのデメリット：キーストロークの多さ

tmuxの最大のデメリットは**キーストローク数の多さ**です。

例えば、新しいタブ（ウィンドウ）を作る操作を比較してみます。

| ツール | 操作 | キーストローク数 |
|--------|------|-----------------|
| WezTerm (macOS) | `Cmd + t` | 2キー同時押し |
| tmux | `Ctrl + b` → `c` | 3キー + 1キー = 4回 |

paneを分割する場合も同様です。

| ツール | 操作 | キーストローク数 |
|--------|------|-----------------|
| WezTerm (macOS) | `Cmd + Shift + Alt + %` | 4キー同時押し |
| tmux | `Ctrl + b` → `%` | 3キー + 2キー = 5回 |

tmuxは「Prefix → コマンド」という2ステップが必要なので、どうしても操作が煩雑になります。WezTermの「同時押し一発」と比べると、この差は無視できません。

## それでもtmuxを選んだ理由

デメリットを理解した上で、それでもtmuxを選びました。理由は**汎用性**です。

- **ターミナル非依存**: WezTermをやめてAlacrittyやKittyに移行しても、同じ操作感
- **サーバー作業**: リモートサーバーでもローカルと同じワークフロー
- **セッション管理**: SSH接続が切れても作業が保持される

特に「ターミナルエミュレータに依存しない」という点が大きい。ツールは乗り換えることがありますが、tmuxの操作を覚えておけば、どの環境でも同じように作業できます。

## 私のtmux設定：デメリットを工夫で解消

tmuxのキーストローク問題は、設定で大幅に改善できます。

### Prefixキーの変更

デフォルトの`Ctrl + b`は押しにくいので、`Ctrl + q`に変更しています。

```bash
unbind C-b
set -g prefix C-q
bind C-q send-prefix
```

### Vim風のキーバインド

pane分割と移動をVim風に設定することで、直感的に操作できるようにしています。

```bash
# pane分割: s で水平、v で垂直
bind s split-window -v
bind v split-window -h

# pane移動: h/j/k/l
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# paneリサイズ: H/J/K/L（リピート可能）
bind -r H resize-pane -L 5
bind -r J resize-pane -D 5
bind -r K resize-pane -U 5
bind -r L resize-pane -R 5
```

これで `Prefix` → `v` でpaneを縦分割、`Prefix` → `h/j/k/l` で移動できます。

### fzfとの連携

セッションやウィンドウの切り替えを、fzfでインタラクティブに行えるようにしています。

```bash
# Prefix + / でセッション一覧をfzfで選択
bind / display-popup -E "tmux list-sessions -F '#{session_name}' | fzf --reverse | xargs tmux switch-client -t"

# Prefix + w でウィンドウ一覧をfzfで選択
bind w display-popup -E "tmux list-windows -a -F '#{session_name}:#{window_index} #{window_name}' | fzf --reverse | cut -d' ' -f1 | xargs tmux switch-client -t"

# Prefix + e で新しいセッション作成
bind e command-prompt -p "new session:" "new-session -s '%%'"
```

セッションやウィンドウが増えてきても、fzfで素早く目的の場所に移動できます。

### その他の便利設定

```bash
# マウス操作を有効化
set -g mouse on

# ウィンドウ番号を1から開始（0始まりより直感的）
set -g base-index 1
setw -g pane-base-index 1

# Escapeキーの遅延をなくす
set -s escape-time 0
```

## まとめ

WezTermのpane機能は十分便利で、正直なところtmuxがなくても困りません。操作も直感的で、覚えることも少ない。

ただ、ターミナルエミュレータに依存した操作を覚えてしまうと、ツールを乗り換えたときに困ります。tmuxを使っていれば、どの環境でも同じワークフローを維持できます。

tmuxのキーストロークの多さは確かにデメリットですが、設定次第で十分に改善できます。Vim風のキーバインドやfzf連携を入れることで、WezTermと遜色ない操作感を実現できました。

「今のツールで満足しているけど、将来のことも考えたい」という方は、tmuxを試してみる価値はあると思います。

## 参考

- [WezTerm公式ドキュメント](https://wezfurlong.org/wezterm/)
- [tmux公式Wiki](https://github.com/tmux/tmux/wiki)
