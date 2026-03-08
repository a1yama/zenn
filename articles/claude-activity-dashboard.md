---
title: 'Claude Codeの活動ログを可視化するダッシュボードを作った'
emoji: '📊'
type: 'tech'
topics:
  - 'ClaudeCode'
  - 'AI'
  - 'Python'
  - 'TypeScript'
  - 'Datasette'
published: false
---

Claude Codeを日常的に使っていると、「自分がどれくらいClaude Codeを使っているのか」「どのツールが多く使われているのか」「どのプロジェクトで活発に使っているのか」が気になってきます。

Claude Codeは `~/.claude/projects/` 配下にJSONL形式で活動ログを保存しています。このログを解析して可視化するダッシュボードを作りました。

https://github.com/a1yama/claude-activity-dashboard

## できること

### ダッシュボード画面

ブラウザで以下の情報を一覧できます。

- **統計カード**: 総セッション数、メッセージ数、ツール使用回数、アクティブプロジェクト数
- **日別アクティビティチャート**: 日ごとのセッション数・メッセージ数の推移
- **時間帯別分布チャート**: 何時にClaude Codeを使っているか
- **ツール使用ランキング**: Bash、Read、Edit等のツール使用頻度
- **プロジェクト別サマリー**: プロジェクトごとの利用状況
- **最近のセッション一覧**: 直近のセッションへのリンク

![最近のセッション一覧](/images/claude-activity-dashboard-sessions.png) _最近のセッション一覧。プロジェクト名・所要時間・メッセージ数・ツール使用回数が一目でわかる_

### セッション詳細画面

個々のセッションを掘り下げて確認できます。

- セッションのメタ情報（プロジェクト名、開始/終了時刻、メッセージ数）
- メッセージ一覧（ユーザー / アシスタントの全文表示）
- ツール使用の詳細（ツール名 + 入力パラメータ）

### 活動ログ分析スキル（`/analyze-usage`）

ダッシュボードに加えて、Claude Code上で `/analyze-usage` スキルを実行すると、活動ログの分析レポートと改善提案を受けられます。

```bash
/analyze-usage              # 全期間分析
/analyze-usage 直近7日      # 期間フィルタ
/analyze-usage dotfiles     # プロジェクト名フィルタ
```

分析内容は以下のカテゴリに分かれています。

| カテゴリ         | 内容                                                   |
| ---------------- | ------------------------------------------------------ |
| ツール効率       | Bashで実行されたが専用ツールで代替可能なコマンドの検出 |
| 指示パターン     | 繰り返し指示の検出、頻出キーワードの自動抽出           |
| エラーパターン   | 修正・やり直し指示、意図ズレの検出                     |
| プロジェクト横断 | プロジェクト別のツール/メッセージ比率                  |
| セッション効率   | 長時間セッションTOP10、時間帯別活動量                  |

分析結果に基づいて、CLAUDE.mdへの追加ルール提案や新規スキル候補の提案も自動生成されます。

## アーキテクチャ

```text
~/.claude/projects/**/*.jsonl
        │
        ▼
   ingest.py ──► SQLite (data/claude_activity.db)
        │
        ▼
   Datasette (JSON API, port 8765)
        │
        ▼
   React + Vite (フロントエンド)
```

技術スタックは以下の通りです。

- **データ取り込み**: Python（`ingest.py`）でJSONLをパースしSQLiteに格納
- **API**: [Datasette](https://datasette.io/) がSQLiteをそのままJSON APIとして公開
- **フロントエンド**: React + TypeScript + Tailwind CSS + Recharts

### なぜDatasetteを使ったのか

Datasetteは、SQLiteファイルを渡すだけでWeb UIとJSON APIを自動生成してくれるツールです。APIサーバーを自前で書く必要がなく、`metadata.yml`にSQLクエリを定義するだけでエンドポイントが生えます。

```yaml
# metadata.yml の一部
queries:
  daily-activity:
    title: 日別アクティビティ
    sql: |
      SELECT
        date_jst AS date,
        COUNT(DISTINCT session_id) AS sessions,
        SUM(CASE WHEN type = 'user' AND is_meta = 0 THEN 1 ELSE 0 END) AS user_messages,
        SUM(tool_count) AS tool_uses
      FROM messages
      GROUP BY date_jst
      ORDER BY date_jst DESC
```

このクエリが `http://localhost:8765/claude_activity/daily-activity.json` として自動的にアクセス可能になります。フロントエンドはこのJSON APIを叩くだけです。

## データ取り込みの仕組み

`ingest.py` がClaude Codeのログファイルを処理します。

Claude Codeのログは `~/.claude/projects/<ディレクトリ名>/` 配下にセッションごとのJSONLファイルとして保存されています。ディレクトリ名はプロジェクトのパスをハイフン区切りに変換したもの（例: `-Users-a1yama-ghq-github-com-foo-bar`）です。

`ingest.py` では以下の処理を行っています。

1. ディレクトリ名からプロジェクト名を復元（`ghq/github.com/foo/bar` のような読みやすい形に変換）
2. 各JSONLファイルを1行ずつパースし、メッセージの種類（user/assistant/system）を判別
3. アシスタントメッセージからツール使用情報（ツール名、入力パラメータ）を抽出
4. セッション単位で集計（メッセージ数、ツール使用回数、開始/終了時刻）
5. SQLiteの `sessions` テーブルと `messages` テーブルに格納

```python
def extract_project_name(dir_name: str) -> str:
    """ディレクトリ名を読みやすいプロジェクト名に変換"""
    parts = dir_name.lstrip("-").split("-")
    for marker in ["ghq", "work"]:
        if marker in parts:
            idx = parts.index(marker)
            result = "/".join(parts[idx:])
            result = re.sub(r"github/com/", "github.com/", result)
            return result
    if parts[0] == "Users" and len(parts) > 2:
        return "/".join(parts[2:])
    return "/".join(parts)
```

## フロントエンド構成

フロントエンドはReact + TypeScriptで構成されています。

```text
frontend/src/
├── pages/
│   ├── Dashboard.tsx        # メインダッシュボード
│   └── SessionDetail.tsx    # セッション詳細
├── components/
│   ├── DailyActivityChart.tsx
│   ├── HourlyDistributionChart.tsx
│   ├── ToolUsageChart.tsx
│   ├── ProjectSummary.tsx
│   ├── RecentSessions.tsx
│   ├── StatCard.tsx
│   ├── MessageList.tsx
│   └── LoadingSpinner.tsx
├── hooks/
│   └── useQuery.ts          # Datasette API呼び出し
└── types/
```

チャートの描画にはRechartsを使っています。Datasette APIのレスポンス形式に合わせたカスタムフック `useQuery` を用意し、各コンポーネントからシンプルにデータを取得できるようにしています。

## ブラウザからのデータ更新

DatasetteのプラグインとしてRefreshエンドポイントを実装しています。ブラウザ上の「更新実行」ボタンを押すと、`ingest.py` が再実行されてデータが最新化されます。

```bash
# コマンドラインからも更新可能
make ingest
```

## セットアップと使い方

```bash
git clone https://github.com/a1yama/claude-activity-dashboard
cd claude-activity-dashboard
make setup  # Python仮想環境 + npm install
make dev    # Datasette API + Vite dev server 起動
```

`make dev` で起動すると、Datasette APIが `localhost:8765` 、フロントエンドが `localhost:5173` で立ち上がります。Viteの設定でAPIリクエストはDatasetteにプロキシされるため、CORSの問題も発生しません。

## まとめ

Claude Codeの活動ログはJSONL形式で保存されており、これを解析することで自分のAI活用パターンを客観的に把握できます。

- どの時間帯に集中して使っているか
- プロジェクトごとの利用バランス
- ツール使用の偏り（Bashに頼りすぎていないか等）
- `/analyze-usage` スキルによる改善提案

Datasetteを使うことでAPI実装を省略でき、SQLクエリを書くだけで新しい分析軸を追加できる点も気に入っています。
