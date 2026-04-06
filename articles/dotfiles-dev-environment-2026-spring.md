---
title: 'dotfilesで管理する開発環境 2026春'
emoji: '🛠️'
type: 'tech'
topics:
  - 'dotfiles'
  - 'tmux'
  - 'Neovim'
  - 'ClaudeCode'
  - 'CLI'
published: false
---

開発環境を晒す記事が流れてくる季節なので自分も乗っかってみる。dotfilesを3年くらい育て続けていて、そろそろ一度棚卸ししたかったのでちょうどいいと思った。

GNU stowで管理しているdotfilesを軸に、ターミナル周り・エディタ・AI活用まで一通り書いてみる。

https://github.com/a1yama/dotfiles

## dotfiles管理 — GNU stow

dotfilesの管理には [GNU stow](https://www.gnu.org/software/stow/) を使っています。stowはシンボリックリンクを作成するツールです。`packages/` 配下にツールごとまとめておくと、ホームディレクトリをクリーンに保ったまま一元管理できます。

### パッケージ構成

現時点で13パッケージ

```text
dotfiles/
└── packages/
    ├── zsh/          # シェル設定
    ├── nvim/         # Neovim
    ├── tmux/         # ターミナルマルチプレクサ
    ├── starship/     # プロンプト
    ├── git/          # Git設定
    ├── lazygit/      # Git TUI
    ├── wezterm/      # ターミナルエミュレータ
    ├── yazi/         # ファイルマネージャ
    ├── claude/       # Claude Code設定・スキル
    ├── life/         # 人生管理CLI
    └── ...
```

各パッケージは、stowでリンクするとホームディレクトリの正しい場所へ配置されます。`packages/starship/.config/starship.toml` なら `~/.config/starship.toml` にリンクされる、という感じ。

```bash
# 個別パッケージの反映
stow -d packages -t ~ zsh

# 全パッケージの一括反映
for dir in packages/*/; do
  stow -d packages -t ~ "$(basename "$dir")"
done
```

新しいマシンのセットアップは、cloneしてstowを実行するだけです。ツールごとにパッケージが分かれているので、特定の設定だけ持っていくこともできます。これが地味に便利で、会社用マシンにはlifeパッケージは入れない、みたいな使い分けをしています。

## ターミナル — WezTerm + tmux

### WezTerm

ターミナルエミュレータは [WezTerm](https://wezfurlong.org/wezterm/) です。Luaで設定を書けるのが気に入っています。設定ファイル自体がプログラムなので、条件分岐やループも使える。

![WezTermのスクリーンショット](/images/dev-environment-2026-spring/wezterm.png) _WezTermの外観_

設定のポイント：

- **フォント**: HackGen Console NF（14pt）— Nerd Fonts対応で各種アイコンが出る
- **透過**: 85%。`macos_window_background_blur` でぼかし効果も入れている。これだけで見た目の満足度がだいぶ上がる
- **タブバー**: 下部配置、1タブのときは非表示。Nerd Fontsのアイコンでタブの装飾もしている
- **キーバインド**: デフォルトを全部無効化して、`keybinds.lua` で独自に定義

WezTermのpane機能は使わず、ペイン管理はtmuxに任せています。以前はWezTermのpaneで満足していたんですが、「WezTermをやめたら操作感ごと失われる」のが嫌でtmuxに移行しました。

https://zenn.dev/a1yama/articles/wezterm-vs-tmux-pane

### tmux

tmuxのprefixは `C-q` にしています。デフォルトの `C-b` は遠すぎるので変えました。

![tmuxのスクリーンショット](/images/dev-environment-2026-spring/tmux.png) _tmuxのステータスバー。セッション名、CPU/メモリ使用率、ネットワーク速度、時刻を表示している_

#### ペイン操作はVimスタイル

```bash
# 分割
bind s split-window -v   # 下に分割
bind v split-window -h   # 右に分割

# 移動: h/j/k/l
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# リサイズ: H/J/K/L（リピート可）
bind -r H resize-pane -L 5
bind -r J resize-pane -D 5
```

#### ステータスバー

Catppuccin風のカラースキームで自作しました。右側にCPU/メモリ使用率、ネットワーク速度、時刻を表示しています。PREFIX押下時やCOPYモード時は左側にモード表示が出る。ステータスバーの見た目にこだわるのは完全に趣味ですが、毎日目に入るものなので気分が上がります。

#### プラグイン

| プラグイン                      | 用途                           |
| ------------------------------- | ------------------------------ |
| tmux-fzf                        | セッション/ウィンドウのfzf検索 |
| tmux-cpu                        | CPU・メモリ使用率の表示        |
| tmux-net-speed                  | ネットワーク速度の表示         |
| tmux-resurrect + tmux-continuum | セッションの自動保存・復元     |

`tmux-resurrect` + `tmux-continuum` はもう手放せません。マシンを再起動しても前回のセッション構成がそのまま復元されます。ウィンドウ数、ペインの分割、カレントディレクトリまで全部。15分ごとに自動保存されるので意識する必要もないです。

#### 日本語ヘルプ

tmuxのキーバインドは覚えきれなかったので、`C-q ?` で日本語の一覧をfzfで出せるようにしました。

```text
s         下に分割
v         右に分割
h/j/k/l   ペイン移動
H/J/K/L   ペインリサイズ
/         セッション切替
a         Claude Code起動
?         このヘルプ
```

今はほぼ見ないですが、新しいキーバインドを追加したときにすぐ確認できるので残してあります。

## シェル環境 — zsh + starship

### zsh

プラグインはHomebrewで入れて `.zshrc` で読み込むだけです。sheldonのようなプラグインマネージャは使っていません。プラグインが3つしかないのでオーバーキル感がある。

```bash
# .zshrc（抜粋）
eval "$(starship init zsh)"
eval "$(zoxide init zsh)"

source $(brew --prefix)/share/zsh-autosuggestions/zsh-autosuggestions.zsh
source $(brew --prefix)/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
```

設定は `.zsh/` 配下にファイル分割して、`.zshrc` からまとめて読み込んでいます。

#### よく使うエイリアス

```bash
# tmux
alias t="tmux"
alias ta="tmux attach"
alias ts="tmux-start"

# git（gitconfigのエイリアスをさらにラップ）
alias g="git"
alias gst="git st"
alias commit="git cm"
alias pushc="git psc"
alias addfzf="git adf"

# ツール置き換え
alias cat="bat"
alias vim="nvim"

# Claude Code
alias ctsp="claude-tmux spawn"
alias ctst="claude-tmux status"
```

gitのエイリアスは `.gitconfig` 側にも定義していて、さらにzsh側でラップしています。二重構造なのは少し微妙ですが、`commit` だけで `git commit -m` まで展開されるのは快適です。

`switch` 関数でfzfを使ったブランチ切り替えもよく使います。ローカル・リモート問わずインクリメンタルに検索して切り替えられる。

### starship

プロンプトは [starship](https://starship.rs/) です。Rust製で速い。

![starshipのスクリーンショット](/images/dev-environment-2026-spring/starship.png) _starshipプロンプト。ディレクトリ、Gitブランチ・ステータス、時刻を表示_

```toml
[character]
success_symbol = "[❯](bold #00ff00)"
error_symbol = "[✘](bold #ff4444)"

[directory]
truncation_length = 10
style = "bold #00FFFF"
truncate_to_repo = false

[git_status]
modified = '[!($count)](green)'
untracked = '[?($count)](red)'
staged = '[++($count)](green)'
```

こだわりはgit_statusです。modified/untracked/stagedの件数を常に表示させています。`git status` を打たなくてもプロンプトを見るだけで差分の有無と数がわかる。これがあるかないかで体感がかなり違います。

## シェルツール

日常的に使っているCLIツールの一覧です。

| 用途 | ツール | 備考 |
| --- | --- | --- |
| ファジーファインダー | [fzf](https://github.com/junegunn/fzf) | あらゆるところで使う |
| ディレクトリ移動 | [zoxide](https://github.com/ajeetdsouza/zoxide) | cdの学習型置き換え |
| catの置き換え | [bat](https://github.com/sharkdp/bat) | シンタックスハイライト付き |
| findの置き換え | [fd](https://github.com/sharkdp/fd) | 高速・直感的 |
| Git TUI | [lazygit](https://github.com/jesseduffield/lazygit) | ステージング・コミットが楽 |
| ファイルマネージャ | [yazi](https://github.com/sxyazi/yazi) | Rust製、高速 |
| リポジトリ管理 | [ghq](https://github.com/x-motemen/ghq) | `~/ghq/` 配下に統一管理 |
| GitHub CLI | [gh](https://github.com/cli/cli) | PR・Issue操作 |
| JSON操作 | [jq](https://github.com/jqlang/jq) |  |

この中で一番使っているのは間違いなくfzfです。単体で使うことはほぼなくて、他のツールと組み合わせることがほとんど。

### fzfの活用

Gitエイリアスにfzfを仕込んで、ファイル選択やブランチ選択をインタラクティブにしています。

```bash
# git add をfzfで（変更ファイルを複数選択してステージング）
adf = "!f() { files=$(git status --short | grep -E \"^( M| M|??)\" | awk \"{print \\$2}\" | fzf --multi); [ -n \"$files\" ] && git add $files; }; f"

# git restore をfzfで
rsf = "!f() { files=$(git status --short | grep -E \"^( M| M| D)\" | awk \"{print \\$2}\" | fzf --multi); [ -n \"$files\" ] && git restore $files; }; f"

# ブランチ削除をfzfで（複数選択可）
deletebranch = "!f() { branches=$(git branch --format=\"%(refname:short)\" | fzf --multi); [ -n \"$branches\" ] && git branch -d $branches; }; f"
```

ポイントは `--multi` で複数選択を有効にしていること。Tabキーで複数ファイルやブランチを選んでまとめて操作できます。一度これに慣れると素の `git add` には戻れません。

### lazygit

lazygitには [git-cz-go](https://github.com/a1yama/git-cz-go)（自作のConventional Commitsツール）を `Z` キーで呼び出せるようにしています。lazygitの画面からそのままコミットメッセージのテンプレートに入れるので、流れが途切れなくて良いです。

```yaml
customCommands:
  - command: git-cz-go
    context: global
    key: Z
    output: terminal
```

### yazi

ファイルマネージャの [yazi](https://github.com/sxyazi/yazi) はNeovimからも呼び出せるようにしています。隠しファイルもデフォルトで表示する設定。正直、最初はいらないかなと思っていたんですが、ディレクトリ構造を眺めながらファイルを開く動線が意外と快適で、今は結構使っています。

## エディタ — Neovim

エディタは [Neovim](https://neovim.io/) です。VSCodeからの移行組で、最初はキーバインドに苦しみました。でも慣れてからはもう戻れなくなった。プラグイン管理は [lazy.nvim](https://github.com/folke/lazy.nvim)。

![Neovimのスクリーンショット](/images/dev-environment-2026-spring/neovim.png) _Neovimの外観_

### プラグイン構成

プラグインは15個くらい。ファイルごとに分けて管理しています。

```text
nvim/lua/plugins/
├── lsp.lua           # LSP設定
├── cmp.lua           # 補完
├── telescope.lua     # ファジーファインダー
├── treesitter.lua    # シンタックスハイライト
├── copilot.lua       # GitHub Copilot
├── claudecode.lua    # Claude Code連携
├── git.lua           # Git統合
├── tree.lua          # ファイルツリー
├── yazi.lua          # yaziとの連携
├── lualine.lua       # ステータスライン
├── toggleterm.lua    # ターミナル
├── whichkey.lua      # キーバインドヘルプ
├── formatter.lua     # フォーマッター
├── colorschema.lua   # カラースキーム
└── nvim-web-devicons.lua
```

LSP + Copilot + Claude Codeの3つが揃っていると、コーディング中のサポートがかなり手厚いです。Copilotがインラインで補完を出しつつ、ファイルをまたぐような大きな変更はClaude Codeに任せる。そういう使い分けをしています。

## バージョン管理 — anyenv

言語のバージョン管理は [anyenv](https://github.com/anyenv/anyenv) です。

```bash
$ anyenv versions
nodenv:
  v23.7.0

pyenv:
  3.13.1

goenv:
  1.26.1
```

nodenv, pyenv, goenvをanyenv経由でまとめて管理しています。miseが良いという話はよく聞くし移行も考えてみましたが、今のところ困っていないのでそのまま。困ってからでいいかなと思っています。

## Git周り — ghq + fzf統合

### ghq

リポジトリの管理は [ghq](https://github.com/x-motemen/ghq) です。すべてのリポジトリが `~/ghq/github.com/<owner>/<repo>/` 配下に配置されます。

```bash
$ ls ~/ghq/github.com/a1yama/
claude-activity-dashboard/
dotfiles/
gh-repo-clean/
git-cz-go/
life/
mcp-radar/
othelp/
tig-gh/
zenn/
```

現時点で自分のリポジトリが9個。ghqのおかげでリポジトリの場所に迷うことがなくなりました。

### fzf統合のGitエイリアス

`.gitconfig` にfzfを組み込んだエイリアスを大量に定義しています。変更ファイルの選択、ブランチの切り替え・削除・マージなど、選択系の操作はすべてfzf経由。

### git-delete-remote-branch

リモートブランチの削除もfzfで選択できる自作スクリプトを使っています。PRがマージされたあとのリモートブランチ掃除が楽になります。

https://zenn.dev/a1yama/articles/git-deleteremotebranch-fzf-gh

## AI活用 — Claude Code + claude-tmux

### Claude Code

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) はターミナルで動くAIコーディングアシスタントです。正直、ここ1年で自分の開発フローに一番インパクトがあったツールだと思います。

dotfilesの `claude` パッケージでCLAUDE.md、settings.json、スキル、コマンドをまとめて管理しています。

```text
packages/claude/
├── .claude/
│   ├── CLAUDE.md        # グローバル指示書
│   ├── settings.json    # 権限設定
│   ├── skills/          # カスタムスキル
│   └── commands/        # カスタムコマンド
└── .local/bin/
    ├── claude-tmux      # 並列エージェント管理CLI
    └── claude-notify    # 完了通知
```

`CLAUDE.md` には「日本語で会話する」「コメントは"なぜ"を書く」みたいなルールを書いています。プロジェクトごとのCLAUDE.mdと組み合わせることで、チームやプロジェクトに合わせた振る舞いをさせられる。

CLAUDE.mdもstowで管理しているのがお気に入りポイントです。どのマシンでも同じルールセットでClaude Codeを使えます。

### claude-tmux — 並列エージェント

claude-tmuxは、tmux上でClaude Codeエージェントを並列実行するための自作CLIです。

```bash
# 新しいtmuxペインでエージェントを起動
claude-tmux spawn "認証APIのテストを書いて" --name auth-test

# 複数のGitHub Issueを並列で処理
claude-tmux issues 42 43 44

# 実行中のエージェント一覧
claude-tmux status
```

tmuxのペイン分割を使って、独立したタスクを複数のエージェントに同時に任せられます。エージェントが質問を持った場合は監督エージェントが自動で判断して回答する仕組みも入れています。

`C-q a` でカレントディレクトリにClaude Codeを即座に起動できるようにもしています。

```bash
# tmux.conf
bind a run-shell 'tmux new-window -n "claude" -c "#{pane_current_path}" "claude"'
```

tmuxとの相性が本当に良いです。Claude Codeを使い始めてからtmuxに移行して正解だったと心から思います。

## GitHubで人生を管理する — life repo

開発環境の話からは少し外れますが、dotfilesの `life` パッケージで管理しているCLIツール群にも触れてみます。

もともとのきっかけはこの記事です。

https://zenn.dev/hand_dot/articles/85c9640b7dcc66

「GitHubで人生を管理する」というコンセプトにすごく共感して、自分も試してみました。最初はObsidianでメモやタスクを管理していたんですが、結局ターミナルで完結させたくなった。[nb](https://github.com/xwmx/nb) を参考にした自作コマンドでGitHubリポジトリへ集約する形に落ち着きました。タスクはGitHub Issues + Projects、メモやノートはMarkdownでリポジトリに保存しています。

### life-note — nb風ノート管理

`life-note`（エイリアス: `ln`）はnbを参考にしたノート管理コマンドです。`.index` ファイルで更新日時順にノートを追跡して、fzfで検索できます。テキストを渡すとClaude APIでタイトルを自動生成してくれる。

```bash
# エディタでノートを作成
ln

# テキストからノートを作成（AIがタイトルを自動生成）
ln "来週のミーティングで話すこと: ..."

# fzfでノート一覧を検索・表示
ln ls
```

### life-memo — クイックメモ

`life-memo` はその日の日次ファイル（`ideas/daily/YYYY-MM-DD.md`）にワンライナーでメモを追記するコマンドです。AI整形で誤字も直してくれます。深夜0時〜6時は前日扱いになる日付境界を入れているのが地味なこだわり。

### life-issue — Issue作成

`life-issue` はGitHub Issueの作成とProjectへの追加をワンコマンドで行います。ラベルはfzfで複数選択、本文はAIが整形してくれる。

### daily-reminder — 定時通知

LaunchAgentで1日4回走る通知スクリプトです。朝10:30にGitHub ProjectsのIn Progress/Todoタスクを通知。夜18:00にはDone未CloseのIssueを自動Closeして、変更をコミット・プッシュします。通知先はDiscordとSlackのWebhook。

Obsidianのときは「あとで整理しよう」が溜まっていく一方でした。CLI投入 → GitHub管理 → 定時通知で棚卸し、というフローにしてからちゃんと回るようになりました。

## Homebrew — Brewfileで一元管理

インストールしているパッケージは `~/.Brewfile` で管理しています。現時点でフォーミュラ242個、Cask38個。`brew bundle dump` で現在の状態をエクスポートして、新しいマシンでは `brew bundle --global` で一括インストールできます。

主要なCask（GUIアプリ）:

| カテゴリ           | アプリ                             |
| ------------------ | ---------------------------------- |
| ブラウザ           | Arc, Google Chrome                 |
| コミュニケーション | Slack, Discord, Zoom               |
| 開発               | Docker, JetBrains Toolbox, Postman |
| デザイン           | Figma                              |
| ユーティリティ     | 1Password, Alfred, CleanShot       |
| ノート             | Notion                             |

## おわりに

こうして整理してみると、fzfとtmuxとClaude Codeの3つが環境全体の軸になっていることに気づきます。fzfはシェルのあらゆる選択操作に、tmuxはウィンドウ管理とセッション永続化に、Claude Codeはコーディングからlife repoの管理まで。この3つを抜かれたら多分仕事にならない。

dotfilesはstowのおかげで「ツールを試す → パッケージに追加 → stow」のサイクルが軽く回せます。新しいツールを見つけるたびにパッケージが増えていくので、今後も育てていくと思います。
