# New Relic と CloudWatch の役割分担

ここまでの章は製品に依存しない一般論だった。この章では、**New Relic（Browser・APM）とAWS CloudWatch Logsを併用する構成**で、どちらに何を任せ、どうつなぐかを整理する。例には[共通例2：予約の問い合わせ](../example-reservation.md)を使い、アプリはGoで書かれたAPIとする。

製品の仕様は2026-10-08に公式ドキュメントで確認した。実環境での動作確認はしていない。

## 役割分担

| | New Relic（Browser・APM） | CloudWatch Logs |
| --- | --- | --- |
| 役割 | 調査の入口と、処理のつながり・性能を見る | 業務上の判断の証跡と、詳細なアプリログを残す |
| 主なデータ | Browserのページ表示・AJAX、APMのトランザクション、分散トレースのspan、エラー | アプリが出す構造化ログ（JSON） |
| 得意なこと | 遅い・失敗したリクエストを見つける。画面→API→DB・外部APIの流れと所要時間を見る | 「その時点で何を判断して返したか」を一件ずつ確かめる |
| 共通例2で置くもの | 予約可能日APIのトランザクション、所要時間、エラー、`reservation_flow_id` などの属性 | `reservation.availability.evaluated`（返した日付と、返さなかった理由）、`reservation.rejected` |
| 既定の保存期間 | 標準の契約（Original Data option）でBrowser・APMのトランザクション・spanは8日、ログは30日 | ロググループごとに設定。既定は無期限 |

保存期間の違いが、役割分担の理由の一つになる。New Relicのトランザクションやトレースは標準の契約では8日で消える（[New Relic: Manage data retention](https://docs.newrelic.com/docs/data-apis/manage-data/manage-data-retention/)）。問い合わせが8日より後に来たら、New Relicからは辿れない。判断の証跡は、既定で無期限に残るCloudWatch Logsに置き（[AWS: Working with log groups and log streams](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html)）、**CloudWatchの記録だけでも調査を始められるよう、必要なIDをログ自体に入れる**。

## 同じログを二重に保存しない

New RelicのGoエージェントは、v3.20.0以降、アプリのログをNew Relicへ転送する機能が**既定で有効**になっている（[New Relic: Configure logs in context for Go](https://docs.newrelic.com/docs/logs/logs-context/configure-logs-context-go/)）。アプリのログをCloudWatchに出している構成でこのままにすると、同じログがNew RelicとCloudWatchの両方に保存される。

ログの保存先をCloudWatchにするなら、Goエージェントは次の設定にする。

- `newrelic.ConfigAppLogForwardingEnabled(false)`：New Relicへのログ転送を止める
- `newrelic.ConfigAppLogDecoratingEnabled(true)`：ログに `trace.id`・`span.id`・`entity.guid`・`entity.name`・`hostname` を付け加える

後者で付く `trace.id` が、New RelicとCloudWatchをつなぐ鍵になる。ログ出力はslog・logrus・zapなら公式のプラグイン（`nrslog`・`nrlogrus`・`nrzap`）を使い、トランザクションの `context` を渡して出す。

反対に、New Relicの画面でログも一緒に見たい場合は、CloudWatch LogsのログをNew Relicへ転送する方法もある（AWS Lambdaの `newrelic-log-ingestion` やAmazon Data Firehoseなど。[New Relic: Forward logs](https://docs.newrelic.com/docs/logs/forward-logs/)）。この場合は両方に保存することになるので、量と費用を比べて決める。

## 二つをつなぐID

| ID | New Relicでの置き場所 | CloudWatchのログ | 役割 |
| --- | --- | --- | --- |
| `trace.id` | 分散トレース・トランザクション（自動） | エージェントの装飾で自動付与 | 画面のAJAXからAPI・DBまでの一回の処理を、両方で同じ値で引く |
| `reservation_flow_id` | トランザクションの属性（`txn.AddAttribute`） | ログのフィールド | 複数のリクエストにまたがる一回の予約の流れをまとめる |
| `session_id` | トランザクションの属性 | ログのフィールド | ログインからのセッションをまとめる |
| `user_ref` | `enduser.id`（Browserの `setUserId`、APMの `txn.SetUserID`） | ログのフィールド | 問い合わせの利用者から記録を探す入口 |

### 画面からAPIへ、トレースを引き継ぐ

New RelicのBrowserエージェントは、同じオリジンへのAJAXリクエストに `newrelic`・`traceparent`・`tracestate` ヘッダーを付け、APMのトレースとつなぐ。APIが別オリジンにある場合は、Browserの設定で対象のオリジンを追加し、API側のCORSで `Access-Control-Allow-Headers` にこの3つのヘッダーを許可する必要がある（[New Relic: Browser data in distributed tracing](https://docs.newrelic.com/docs/browser/new-relic-browser/browser-pro-features/browser-data-distributed-tracing/)）。

### 業務上のIDを、両方に載せる

`trace.id` は一回のリクエストの単位なので、予約の流れ全体を引くには業務上のIDも要る。APIで受け取った `reservation_flow_id` などを、New Relicのトランザクション属性とCloudWatchのログの両方に入れる。

```go
txn := newrelic.FromContext(ctx)
txn.AddAttribute("reservation_flow_id", flowID)
txn.SetUserID(userRef) // enduser.id になる。メールアドレスや患者番号は入れない
```

Browser側でも、`newrelic.setCustomAttribute("reservation_flow_id", flowID)` で、その後のAJAXやページ表示のイベントに同じ値を付けられる（[setCustomAttribute](https://docs.newrelic.com/docs/browser/new-relic-browser/browser-apis/setcustomattribute/)）。画面でログを出していなくても、Browserエージェントを入れていれば使える。

`enduser.id` には `user_ref` のような内部の参照用IDを入れる。New Relicは外部のサービスなので、メールアドレスや患者番号を送らない（[情報を守る](../security-and-privacy.md)）。

## Goエージェントは、自分で計装する

Goはコンパイルされたバイナリで動くため、New RelicのGoエージェントは自動では計装されない。何を計測するかをコードで書く必要がある（[New Relic: Guide to using the Go agent API](https://docs.newrelic.com/docs/apm/agents/go-agent/api-guides/guide-using-go-agent-api/)）。

| 計測したいもの | 書くこと |
| --- | --- |
| HTTPリクエストをトランザクションにする | ハンドラーを `newrelic.WrapHandleFunc` などで包む |
| DBの呼び出し | DatastoreSegment、またはドライバー用の統合パッケージ |
| 外部APIの呼び出し | `newrelic.StartExternalSegment`。分散トレースの引き継ぎにも使われる |
| goroutineでの処理 | `txn.NewGoroutine()` で渡す |
| 業務上の属性 | `txn.AddAttribute` |

エラーはエージェントが自動で検知する（同ガイド）。「技術的なコンテキストは自動、業務上の証跡は明示」という方針に当てはめると、上の表の前の4行をミドルウェアや共通処理に寄せ、業務上の属性と判断の証跡をアプリのコードで明示的に出す形になる。

## 問い合わせから調べる手順

[共通例2](../example-reservation.md)の「10月10日が選べなかった」という問い合わせを例にする。

### 8日以内なら、New Relicから入る

1. New Relicで利用者（`enduser.id`）と日時から、Browserのページ表示や予約可能日APIのトランザクションを探す
2. トランザクションの所要時間やエラーを見て、遅延や障害がなかったかを確かめる
3. トランザクションの `trace.id` と `reservation_flow_id` を控える

```sql
SELECT timestamp, trace.id, reservation_flow_id, duration
FROM Transaction
WHERE enduser.id = 'u-7d41' AND name LIKE '%available%'
SINCE '2026-10-03 09:00:00' UNTIL '2026-10-03 10:00:00'
```

### 次に、CloudWatch Logs Insightsで判断の証跡を見る

CloudWatch Logs InsightsはJSONのフィールドを自動で認識し、入れ子のフィールドは `unavailable_by_reason.slot_full` のようにドットでつないで扱える（[AWS: Supported logs and discovered fields](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_AnalyzeLogData-discoverable-fields.html)）。

```text
fields @timestamp, event, evaluated_at, available_dates, unavailable_by_reason.slot_full, rule_version
| filter reservation_flow_id = "rf-3301"
| sort @timestamp asc
```

8日を過ぎてNew Relicに記録が残っていない場合は、`reservation_flow_id` の代わりに `user_ref` と日時で同じ検索から始める。判断の証跡にIDを入れておいたので、New Relicがなくても調べられる。

## 未確認のこと

- 上記の設定やクエリを、実際の環境で動かしてはいない。エージェントのバージョン、ログライブラリ、契約内容によって挙動や保存期間は変わる
- New Relic側の保存期間は契約や組織の設定で変わる。自分の環境の値を確かめる
- New RelicとCloudWatchそれぞれの費用は見積もっていない
