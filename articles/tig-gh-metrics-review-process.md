---
title: 'tig-ghのメトリクス機能でPRレビュープロセスを可視化・改善する'
emoji: '📊'
type: 'tech'
topics:
  - 'GitHub'
  - 'PR'
  - 'レビュー'
  - 'メトリクス'
  - 'Go'
published: false
publication_name: 'kyash'
---

KyashでBackend Engineering Managerをしている[a1yama](https://x.com/xxx_a1_xxx)です。
この記事は [Kyash Advent Calendar 2025](https://adventar.org/calendars/11602) の5日目の記事です。

## はじめに

チーム開発をしていく上で、PRの書きぶりが人によって違っていたり、PRが長時間放置されたりと、レビュープロセスやリリース頻度に課題を感じることがありました。

もともと個人でGitHubの内容をターミナルで気軽に見れる tig-gh というツールの作成をしていたのですが、上記の課題の解決をしたく、メトリクス機能を追加しました。本記事では、メトリクス機能でレビュープロセスを可視化し、開発フローを改善する方法を紹介します。

## tig-gh

tig-ghは、tigライクな操作感でGitHubを管理できるTUIツールです。Issue、Pull Request、Commitなどをターミナルから快適に操作できます。

https://github.com/a1yama/tig-gh

### 主な特徴

- **tigライクな操作感**: `j`/`k`でリスト移動、`Enter`で詳細表示など、tigユーザーに馴染みのあるキーバインディング
- **複数ビュー対応**: Issues、Pull Requests、Commits、Search、Review Queue、Metricsの各ビュー
- **高速なキャッシュ機構**: GitHub API呼び出し結果のメモリ＋ファイルキャッシュ
- **(New!!) メトリクス機能**: 複数リポジトリのレビュープロセス分析

本記事では、特に**メトリクス機能**にフォーカスします。

## メトリクス機能の詳細

メトリクスビューでは、複数リポジトリのレビュープロセスを多角的に分析できます。以下は実際の表示例です（googleのOSSリポジトリを対象にした例）。

### 1. Overall Lead Time / Review Phase Breakdown

```
 Lead Time Metrics
Period: 2025-11-20 ~ 2025-12-04 (14 days)
Last updated: 2025-12-04 10:28:13

 Overall Lead Time
Average: 2d 16h 28m  Median: 1d 5h 20m  PRs: 42

 Review Phase Breakdown
  PR Created → First Review:     avg 19h 18m (8 PRs)
  First Review → Approval:       avg 2d 10m (8 PRs) ← ボトルネック
  Approval → Merge:              avg 1d 3h 42m (8 PRs)
  ─────────────────────────────────────────────
  Total Lead Time:               avg 3d 23h 11m
```

レビュープロセスを「PR作成 → 初回レビュー → 承認 → マージ」のフェーズに分解し、各フェーズの平均所要時間を可視化します。最も時間がかかっているフェーズには「← ボトルネック」と表示されるので、改善ポイントが一目でわかります。

リードタイムの短縮は、マージ頻度の向上、ひいてはリリース頻度の向上に直結します。

### 2. Activity by Day of Week

```
 Activity by Day of Week
         Mon  Tue  Wed  Thu  Fri  Sat  Sun
Merges     5   8  21   6   2   0   0
Reviews    5   0   1   1   0   0   1
```

曜日ごとのレビュー数とマージ数を可視化します。この例では水曜日のマージが突出して多いことがわかります。金曜日のマージが多い場合は週末前のマージを避けるルールを検討するなど、デプロイ戦略やレビュー体制の最適化に活用できます。

### 3. Weekly Comparison

```
 Weekly Review Activity (This Week vs Last Week)
Period                       Reviews     Merges
This Week (last 7 days)            3         19
Last Week (8-14 days ago)          3         23
Change                         +0.0%     -17.4%
```

今週と先週の主要メトリクスを比較し、増減率を表示します。週次でメトリクスの変化を追跡して、改善トレンドを確認できます。

### 4. PR Quality Issues

```
 PR Quality Issues (5 issues)
High Priority:
Repo                            #       Type             Details          Title
google/closure-templates        #205    no_description   0 lines, 0 files Fix "type expression reference" link URL
google/closure-templates        #206    no_description   0 lines, 0 files fix link url to plugins.md
google/closure-templates        #217    no_description   0 lines, 0 files Fixing invalid links
google/guava                    #3617   no_description   0 lines, 0 files fix duplicate words typos in comments
google/guava                    #4034   no_description   0 lines, 0 files google#1409 Add poll() and offer()
```

PRの品質に関する問題を自動検出します。大規模PR、説明不足、レビュアー不足などを検出し、品質問題のあるPRを早期に発見できます。

### 5. Stagnant PRs

```
 Stagnant PRs (Open > 3d)
Total stagnant PRs:  235
Longest waiting PRs:
   1. google/guava #2100 (3801d): UnsignedShort patch as pull request
   2. google/guava #2148 (3744d): byte array searches with start and end limits
   3. google/guava #2180 (3709d): Add tryParse methods to UnsignedBytes...
   4. google/guava #2210 (3690d): Add Futures.getUnchecked with timeout
   5. google/guava #2242 (3654d): preserve the laziness of incoming iterators
```

3日以上オープンになっているPRを「滞留PR」として検出します。定期的に確認して、放置PRにアクションを促せます。

### 6. Per Repository

```
 Per Repository
Repository                                        Avg       Median    PRs
google/closure-templates                   1d 16h 54m      11h 56m      6
google/go-github                           3d 23h 11m   1d 13h 36m      8
google/guava                                      44m          27m      4
google/meridian                            2d 22h 45m   1d 13h 56m     24
```

各リポジトリの平均リードタイム、中央値、PR数を表示します。リポジトリ間のパフォーマンスを比較して、改善が必要なリポジトリを特定できます。リードタイムが短いリポジトリの良いプラクティスを他リポジトリに展開するといった活用ができます

## 設定方法

メトリクス機能を使用するには、`~/.config/tig-gh/config.yaml` に対象リポジトリと計測期間を指定します。

```yaml:~/.config/tig-gh/config.yaml
github:
  token: ghp_xxxxxxxxxxxx
  repositories:
    - your-org/repo1
    - your-org/repo2
    - your-org/repo3

metrics:
  enabled: true
  lead_time_enabled: true
  calculation_period: 720h  # 30日間（720h=30日, 2160h=90日）
  show_review_phases: true
  show_day_of_week: true
  show_weekly_comparison: true
  show_quality_issues: true
  show_stagnant_prs: true
  show_repository_stats: true
```

## まとめ

tig-ghのメトリクス機能を活用することで、以下のような改善に繋げられます。

1. **レビュープロセスの可視化**: どこがボトルネックかを定量的に把握
2. **滞留PRの早期発見**: 放置されているPRを定期的に確認
3. **PR品質の向上**: 大規模PRや説明不足PRを自動検出
4. **曜日パターンの把握**: デプロイ戦略やレビュー体制の最適化
5. **継続的な改善**: 週次比較で改善トレンドを追跡

メトリクスは「測定すること」が目的ではなく、「改善すること」が目的です。tig-ghのメトリクス機能を使って、チームのレビュープロセスを継続的に改善できます。

ターミナルからサクサクとGitHubを操作しながら、チームのメトリクスを確認できます。

個人開発でありながらKyashのバックエンドに合わせた要件定義にしているため、万人向けを想定してはいません。チームメンバーからは「計測ブランチを絞りたい」「もっと気軽に見れるようにしたい」といった意見をもらっており、改善を進めています。今後はメトリクスの定期Slack通知なども検討中です。

チーム定例でもこのツールを使ってPRの整理や状態の確認を行い、レビュープロセスの最適化とリリース頻度の向上を目指しています。

フィードバックやコントリビューションをお待ちしています！

https://github.com/a1yama/tig-gh

ご質問やフィードバックがあれば、GitHubのIssueやXでお気軽にご連絡ください。

それでは、良いレビューライフを！
