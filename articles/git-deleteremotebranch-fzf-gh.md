---
title: 'fzf + gh CLIでリモートブランチを安全に一括削除するGitエイリアスを作った'
emoji: '🧹'
type: 'tech'
topics:
  - 'Git'
  - 'GitHub'
  - 'fzf'
  - 'ShellScript'
  - 'CLI'
published: true
---

## リモートブランチ、溜まっていませんか？

プロジェクトが長く続くと、リモートブランチが際限なく増えていきます。自分のプロジェクトでも気づいたら相当な数が溜まっていました。

本来はPRマージ時にブランチを削除するのが一番です。GitHubにも「Automatically delete head branches」という設定があり、マージ後にブランチを自動削除してくれます。

ただ現実には、こうした設定が入っていないリポジトリを途中から引き継ぐこともあれば、設定を入れる前に溜まったブランチはそのまま残り続けます。PRを経由せず直接pushされたブランチや、作りかけで放置されたブランチも自動削除の対象にはなりません。結果として、気づいたときには手作業で掃除するしかない状態になりがちです。

そしていざ掃除しようとすると、`git push origin --delete` にはローカルの `git branch -d` のような安全弁がありません。

- 未マージのブランチも無警告で消える
- open PRのブランチを消すとPRが自動closeされてしまう

大量のブランチを1本ずつ確認して消すのも現実的ではないので、安全に一括削除できるツールを作りました。

## 作ったもの

`git deleteremotebranch` というGitエイリアスです。実行すると以下の流れで動作します。

1. `git fetch --prune` でリモートと同期
2. 各ブランチのマージ状態・PR状態を自動で調査
3. fzfで状態ラベル付きの一覧を表示し、複数選択可能
4. ブランチの状態に応じて、削除・確認・スキップを自動判定

### fzfの選択画面

![fzfの選択画面](/images/git-deleteremotebranch-fzf.png)

各ブランチにマージ状態（`[merged]`/`[unmerged]`）とPR状態（`[PR #xxx MERGED]`等）のラベルが付いています。Tabキーで複数選択し、Enterで削除します。

## コード全体

https://github.com/a1yama/dotfiles/blob/main/packages/git/.local/bin/git-delete-remote-branch

以降、ポイントごとに解説していきます。

## ポイント1: git fetch --prune による事前同期

```bash
git fetch --prune >/dev/null 2>&1
```

リモートで既に削除されたブランチがローカルの追跡情報に残っていると、削除時に「remote ref does not exist」エラーになります。開発中に実際にこの問題が発生しました（チームメンバーがリモートブランチを先に削除していた）。

スクリプト冒頭で `git fetch --prune` を実行することで、最新のリモート状態に同期してからブランチ一覧を取得します。

## ポイント2: マージ済みブランチの一括取得で高速化

```bash
# ループの外で1回だけ取得
merged_branches=$(git branch -r --merged "origin/${main_branch}" 2>/dev/null | sed 's/^[[:space:]]*//' || true)

# ループ内はgrepで判定
if echo "$merged_branches" | grep -q "origin/${short}$"; then
    merge_status="[merged]"
fi
```

素朴に実装すると、ループ内で毎回 `git branch -r --merged` を実行してしまいがちです。ブランチ数が多いとこれだけで大きなオーバーヘッドになります。

ループの外で一度だけマージ済みブランチの一覧を取得し、ループ内では `grep` で判定するようにしました。

## ポイント3: 進捗表示

```bash
printf "\r\033[Kブランチ情報を取得中... (%d/%d) %s" "$current" "$branch_count" "$short" >&2
```

ブランチ数が大量にあると、`gh pr list` のAPI呼び出しにかなりの時間がかかります。進捗表示がないと処理が固まったように見えるため、`\r\033[K`（キャリッジリターン + 行クリア）を使って同一行に進捗を上書き表示しています。

stderrに出力しているのは、stdoutはfzfへのパイプに使っているためです。

## ポイント4: 3段階の安全弁

このツールの核となる部分です。ブランチのマージ状態とPR状態の組み合わせに応じて、削除時の挙動を3段階に分けています。

| マージ状態 | PR状態 | 動作                         |
| ---------- | ------ | ---------------------------- |
| merged     | no PR  | そのまま削除                 |
| merged     | MERGED | そのまま削除                 |
| unmerged   | MERGED | そのまま削除（squash merge） |
| unmerged   | CLOSED | ⚠️ 確認プロンプト            |
| unmerged   | OPEN   | ⏭ 自動スキップ              |
| unmerged   | no PR  | ⚠️ 確認プロンプト            |

### open PRのブランチは自動スキップ

```bash
if echo "$line" | grep -qi "\[PR #[0-9]* OPEN\]"; then
    echo "⏭  スキップ: ${branch_name} (open PRがあります)"
    continue
fi
```

open PRがあるブランチを削除するとPRが自動closeされてしまいます。fzfの一覧には表示しつつも（状態が確認できるように）、選択しても削除はスキップします。

### 要注意ブランチは確認プロンプト

「未マージ & PRがclose」「未マージ & PRなし」の場合は、本当に消してよいか `(y/N)` で確認します。デフォルトは `N`（削除しない）なので、安全側に倒れます。

削除対象のブランチ名と理由は赤太文字（`\033[1;31m`）で表示し、ターミナル上で視認しやすくしています。大量のブランチを処理する中で、どのブランチについて判断を求められているのかが一目でわかるようにするためです。

### squash mergeへの対応

GitHubのsquash mergeを使っていると、PRはMERGEDなのに `git branch -r --merged` では未マージと判定されます。このケースは`[unmerged] + [PR MERGED]` という組み合わせになりますが、PRがマージ済みなので安全に削除できます。

## ポイント5: メインブランチの自動検出・除外

```bash
main_branch=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo "main")
```

`git symbolic-ref` でリポジトリのデフォルトブランチを自動検出し、一覧から除外しています。main、master、developなど、リポジトリごとの設定に対応できます。

## ポイント6: gh CLIのjqフィルタでnull対策

```bash
pr_info=$(gh pr list --head "$short" --state all --json number,state \
    --jq 'if length > 0 then "#\(.[0].number) \(.[0].state)" else "" end' 2>/dev/null || echo "")
```

PRが存在しないブランチでは `gh pr list` の結果が空配列になり、`.[0]` は `null` を返します。`if length > 0 then ... else "" end` で空文字を返すようにしてエラーを防いでいます。

## セットアップ

### 前提条件

- [fzf](https://github.com/junegunn/fzf)
- [gh CLI](https://cli.github.com/)（認証済み）

### スクリプトの配置

上記のスクリプトを `git-delete-remote-branch` という名前で保存し、パスの通った場所に配置します。

```bash
# 例: ~/.local/bin/ に配置
cp git-delete-remote-branch ~/.local/bin/
chmod +x ~/.local/bin/git-delete-remote-branch
```

Gitは `git-<name>` という命名のスクリプトをパスから自動的に見つけて `git <name>` として実行できるため、これだけで `git delete-remote-branch` として使えるようになります。

### エイリアスの設定（任意）

より短い名前で呼びたい場合は、Gitエイリアスを設定します。

```ini
[alias]
    deleteremotebranch = !git-delete-remote-branch
```

これで `git deleteremotebranch` で実行できます。

### dotfiles + GNU stowで管理する場合

自分は[dotfiles](https://github.com/a1yama/dotfiles)をGNU stowで管理しています。stowを使うと、パッケージごとにディレクトリを分けつつシンボリックリンクで`$HOME`に展開できるので、こういったスクリプトの管理にも便利です。

```text
packages/git/
├── .config/git/config.d/alias.conf      # エイリアス定義
├── .local/bin/git-delete-remote-branch  # スクリプト本体
└── .gitconfig                           # include設定
```

`stow -v git` で `~/.local/bin/git-delete-remote-branch` にシンボリックリンクが作られます。

ソースコードは以下のリポジトリで公開しています。

https://github.com/a1yama/dotfiles

## ローカル版エイリアスとの比較

ローカルブランチ用のエイリアスも併せて紹介します。

```ini
[alias]
    deletebranch = "!f() { branches=$(git branch --format=\"%(refname:short)\" | fzf --multi); [ -n \"$branches\" ] && git branch -d $branches; }; f"
    deletebranchf = "!f() { branches=$(git branch --format=\"%(refname:short)\" | fzf --multi); [ -n \"$branches\" ] && git branch -D $branches; }; f"
    deleteremotebranch = !git-delete-remote-branch
```

|                | `deletebranch` (ローカル)    | `deleteremotebranch` (リモート) |
| -------------- | ---------------------------- | ------------------------------- |
| 対象           | ローカルブランチ             | リモートブランチ                |
| マージチェック | `git branch -d` が自動で行う | スクリプトで明示的に表示        |
| PRチェック     | なし                         | gh CLIで取得・表示              |
| 強制削除版     | `deletebranchf` (`-D`)       | なし（安全重視）                |
| 実装           | git aliasのワンライナー      | 外部シェルスクリプト            |

ローカル版は `git branch -d` が安全弁になるのでワンライナーで十分ですが、リモート版は安全弁を自前で用意する必要があるため外部スクリプトとして実装しています。リモート版に強制削除オプションを用意していないのも、安全重視の設計意図です。

## まとめ

`git push origin --delete` には安全弁がないという問題に対して、fzfとgh CLIを組み合わせたインタラクティブなツールを作りました。

- マージ状態とPR状態を**視覚的に確認**できる
- open PRのブランチは**自動スキップ**で誤操作を防止
- 要注意ブランチは**確認プロンプト**でデフォルトNo
- squash mergeのケースも考慮

大量のリモートブランチを安全に掃除できました。同じ課題を抱えている方の参考になれば幸いです。
