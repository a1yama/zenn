---
title: 'GoのOpenTelemetry計装が辛すぎるので、ボイラープレートを消すライブラリを作った'
emoji: '🔭'
type: 'tech'
topics:
  - 'Go'
  - 'OpenTelemetry'
  - 'Observability'
  - 'OSS'
published: true
---

## TL;DR

GoのOpenTelemetry計装で最も多いバグ――`RecordError`と`SetStatus`の呼び忘れ――を構造的に排除するライブラリ **[othelp](https://github.com/a1yama/othelp)** を作りました。

```go
// Before: 毎回エラー処理を書く必要がある
ctx, span := otel.Tracer("myapp").Start(ctx, "GetUser")
defer span.End()
if err != nil {
    span.RecordError(err)                    // 忘れがち
    span.SetStatus(codes.Error, err.Error()) // 忘れがち
    return nil, err
}

// After: defer end(&err) で全部やってくれる
ctx, end := tracer.Start(ctx, "GetUser")
defer end(&err)
```

## OpenTelemetryとは

OpenTelemetry（OTel）は、アプリケーションの**トレース・メトリクス・ログ**を収集するためのオープンソースの標準規格です。CNCFのプロジェクトとして開発されており、Datadog、Grafana、New Relicなど主要なObservabilityツールが対応しています。

分散システムにおいて「このリクエストはどのサービスを通って、どこで何ms かかったか」を可視化するために、各関数やサービスに**計装（instrumentation）**と呼ばれるコードを仕込みます。

## GoでOTelはどこで使うのか

GoはマイクロサービスやバックエンドのAPIサーバーで多く採用されています。OTelの計装が特に効果を発揮するのは以下のようなケースです。

- **HTTPハンドラ** — リクエスト単位でトレースを開始し、処理の流れを追跡する
- **DBアクセス** — クエリの実行時間やエラーを記録し、ボトルネックを特定する
- **外部API呼び出し** — 他サービスへのHTTP/gRPCリクエストのレイテンシとエラー率を可視化する
- **バッチ処理** — ジョブの各ステップの進捗と所要時間を記録する

つまり、**Goで書かれるほぼすべてのバックエンドコードにおいて、OTelの計装は必要になる**と言っても過言ではありません。だからこそ、計装コードのBoilerplateが多いのは深刻な問題です。

## GoのOTel計装、何が辛いのか

GoでOpenTelemetryを導入したことがある方なら、こんな経験があるはずです。

### 1. エラー記録のボイラープレートが膨大

関数ごとに同じパターンを繰り返す必要があります。

```go
func GetUser(ctx context.Context, id string) (*User, error) {
    ctx, span := otel.Tracer("myapp").Start(ctx, "GetUser")
    defer span.End()
    span.SetAttributes(attribute.String("user.id", id))

    user, err := db.FindUser(ctx, id)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return nil, err
    }

    span.SetStatus(codes.Ok, "")
    return user, nil
}
```

1つの関数でこれです。プロジェクト全体で数十〜数百の関数に同じパターンを書くことになります。

### 2. RecordError / SetStatus の呼び忘れが頻発する

これが最も深刻な問題です。`defer span.End()` は書くのに、エラー時の `RecordError` と `SetStatus` を忘れる。結果、トレースは出るのにエラー情報が欠落している――という状態になります。

レビューで毎回指摘するのも非現実的です。人間が忘れる問題は、コードの構造で解決すべきです。

### 3. Java/Pythonとの格差

JavaやPythonにはOTelの自動計装（auto-instrumentation）があり、コードを変更せずにトレースを取得できます。Goにはその仕組みがありません。すべて手動です。

## othelp の設計思想

この問題を解決するために、3つの原則でライブラリを設計しました。

### 原則1: `defer end(&err)` パターンが核

Goの名前付き戻り値とdeferを組み合わせることで、エラー記録を構造的に強制します。

```go
func GetUser(ctx context.Context, id string) (user *User, err error) {
    ctx, end := tracer.Start(ctx, "GetUser",
        othelp.Str("user.id", id),
    )
    defer end(&err) // ← errの値を見て自動でRecordError + SetStatus

    user, err = db.FindUser(ctx, id)
    if err != nil {
        return nil, err
    }
    return user, nil
}
```

`end` はerrのポインタを受け取り、deferで関数終了時に評価します。errがnilでなければ `RecordError` + `SetStatus(Error)`、nilなら `SetStatus(Ok)` を自動で行います。

**忘れようがない**のがポイントです。

### 原則2: OTel公式APIを隠蔽しすぎない

「薄いヘルパー」に徹しています。独自のtracer抽象やspan抽象は作りません。必要ならいつでもOTelの公式APIに戻れます。

```go
// escape hatch: 生のOTel Tracerを取り出せる
otelTracer := tracer.OTelTracer()
```

### 原則3: 最小依存

OTel公式SDK以外の外部依存はゼロです。

## 使い方

### インストール

```bash
go get github.com/a1yama/othelp
```

### 初期化

OTelの初期化は通常30〜50行必要ですが、othelpでは数行です。

```go
shutdown, err := othelp.Init(ctx, othelp.Config{
    ServiceName: "myapp",
    Exporter:    "otlp",         // "otlp" or "stdout"
    Endpoint:    "localhost:4317",
    Insecure:    true,
})
if err != nil {
    log.Fatal(err)
}
defer shutdown(ctx)
```

### Tracerの作成

パッケージレベルで宣言して使い回します。

```go
var tracer = othelp.NewTracer("myapp/usecase")
```

### 関数の計装

```go
func CreateOrder(ctx context.Context, req OrderRequest) (order *Order, err error) {
    ctx, end := tracer.Start(ctx, "CreateOrder",
        othelp.Str("order.type", req.Type),
        othelp.Int("order.items", len(req.Items)),
    )
    defer end(&err)

    order, err = db.InsertOrder(ctx, req)
    if err != nil {
        return nil, fmt.Errorf("insert order: %w", err)
    }
    return order, nil
}
```

### 属性ヘルパー

`attribute.String(...)` の代わりに短縮形が使えます。

```go
othelp.Str("key", "value")      // attribute.String
othelp.Int("key", 42)           // attribute.Int
othelp.Float64("key", 3.14)     // attribute.Float64
othelp.Bool("key", true)        // attribute.Bool
othelp.Strs("key", []string{})  // attribute.StringSlice
othelp.Ints("key", []int{})     // attribute.IntSlice
```

## Before / After 比較

### Before（素のOTel）

```go
func ProcessPayment(ctx context.Context, payment Payment) (*Result, error) {
    ctx, span := otel.Tracer("payment-service").Start(ctx, "ProcessPayment")
    defer span.End()
    span.SetAttributes(
        attribute.String("payment.id", payment.ID),
        attribute.Float64("payment.amount", payment.Amount),
        attribute.String("payment.currency", payment.Currency),
    )

    result, err := gateway.Charge(ctx, payment)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return nil, fmt.Errorf("charge failed: %w", err)
    }

    if err := db.SaveResult(ctx, result); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return nil, fmt.Errorf("save result: %w", err)
    }

    span.SetStatus(codes.Ok, "")
    return result, nil
}
```

### After（othelp）

```go
func ProcessPayment(ctx context.Context, payment Payment) (result *Result, err error) {
    ctx, end := tracer.Start(ctx, "ProcessPayment",
        othelp.Str("payment.id", payment.ID),
        othelp.Float64("payment.amount", payment.Amount),
        othelp.Str("payment.currency", payment.Currency),
    )
    defer end(&err)

    result, err = gateway.Charge(ctx, payment)
    if err != nil {
        return nil, fmt.Errorf("charge failed: %w", err)
    }

    if err = db.SaveResult(ctx, result); err != nil {
        return nil, fmt.Errorf("save result: %w", err)
    }

    return result, nil
}
```

エラーリターンが2箇所ある場合、素のOTelでは `RecordError` + `SetStatus` を**4行 × 2箇所 = 8行**書く必要があります。othelpでは **`defer end(&err)` の1行**で済みます。

## 仕組み

`end` 関数の内部実装はシンプルです。

```go
func newEndFunc(span trace.Span) EndFunc {
    return func(errPtr *error) {
        if errPtr != nil && *errPtr != nil {
            span.RecordError(*errPtr)
            span.SetStatus(codes.Error, (*errPtr).Error())
        } else {
            span.SetStatus(codes.Ok, "")
        }
        span.End()
    }
}
```

Goの `defer` は関数終了時に実行され、名前付き戻り値のポインタを通じて最終的なerrorの値を参照できます。この言語仕様を活用しているだけなので、マジックは一切ありません。

## まとめ

| 観点 | 素のOTel | othelp |
|---|---|---|
| エラー記録 | 手動（忘れやすい） | 自動（`defer end(&err)`） |
| 属性セット | `attribute.String(...)` | `othelp.Str(...)` |
| 初期化 | 30〜50行 | 5行 |
| OTel APIへのアクセス | 直接 | `OTelTracer()` で取得可能 |
| 外部依存 | - | OTel SDK のみ |

GoでOTelを使っていて「ボイラープレートが多すぎる」「RecordErrorを忘れてエラーが見えない」と感じている方は、ぜひ試してみてください。

https://github.com/a1yama/othelp
