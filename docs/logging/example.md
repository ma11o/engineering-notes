# 共通例1：レポート出力ジョブ

各章はこの一つの処理を主な例に使う。利用者からの問い合わせを入口に、システムの判断を確かめる例は[共通例2：予約の問い合わせ](example-reservation.md)にある。ここで4つの場面の記録を先に眺めておくと、各章の説明がどの行の話か追いやすい。

この例は説明用に作ったもので、書籍 *Observability Engineering* の事例ではない。組み立てには、書籍の次の考え方を使っている。

- キューの送信側でコンテキストを載せ、受信側で取り出す（第7章 pp.116、119）
- キューへの投入と取り出しは非同期の境界なので、spanにする候補になる（第7章 p.117）
- 処理全体の要約イベントと子の呼び出しを分けて集計する（第6章 p.110）
- 長い処理は中間状態を残し、完了時に要約する（第7章 pp.121–122）

イベント名・フィールド名と、後半のレベル・処理結果・通知の判断表は、書籍にない本ノート独自の提案である。

## 処理の流れ

利用者が月次レポートのPDF出力を依頼する。

1. **API** が `POST /exports` を受け付け、ジョブを作ってキューへ入れ、`202 Accepted` を返す
2. **ワーカー** がキューからジョブを取り出し、外部のPDF生成サービスを呼ぶ。タイムアウトしたら最大3回まで試行する
3. 生成したPDFをストレージへ保存し、DBのジョブ状態を `done` にコミットする
4. 利用者は画面でジョブの状態を見てPDFを取得する

## 使うIDと出来事

| ID | 何が同じなら同じ値か | 生成する場所 |
| --- | --- | --- |
| `request_id` | 受け付けた一回のHTTP呼び出し | API |
| `job_id` | キュー待ち・再試行を含むジョブ全体 | API（キューのメッセージで運ぶ） |
| `attempt` | ワーカーがPDF生成サービスを呼んだ何回目か | ワーカー |
| `trace_id` | 一つのトレース。API→キュー→ワーカーへ引き継ぐ | 最初に受けたAPI |

| 出来事（`event`） | いつ出すか | 記録する主な値 |
| --- | --- | --- |
| `export.accepted` | ジョブをDBへ登録し、キューへ入れた後 | `report_type`、`request_id`、`job_id` |
| `export.started` | ワーカーがジョブを取り出した時 | `queue_wait_ms` |
| `renderer.attempt_completed` | PDF生成サービスの呼び出し一回が終わった時 | `attempt`、`outcome`、`error_code`、`duration_ms` |
| `export.completed` | DBの状態をコミットした後。成否を問わず一件 | `outcome`、`attempt_count`、`recovered`、`page_count`、`size_bytes`、`duration_ms` |

`export.completed` は「成否にかかわらずジョブが終わった」記録で、成功は `outcome` で表す。PDFの中身や宛先のメールアドレスは記録せず、ページ数とサイズで代える。

## 4つの場面

以下では、共通する `trace_id`・`service_version` などを省略する。

### A. 正常終了

```text
INFO  event=export.accepted   request_id=req-7f3a job_id=job-1024 report_type=monthly
INFO  event=export.started    job_id=job-1024 queue_wait_ms=120
DEBUG event=renderer.attempt_completed job_id=job-1024 attempt=1 outcome=success duration_ms=640
INFO  event=export.completed  job_id=job-1024 outcome=success attempt_count=1
      recovered=false page_count=12 size_bytes=284000 duration_ms=780
```

`export.completed` の一行で、ジョブ全体の成否・時間・規模が分かる。遅い例や失敗例と、同じ `report_type`・`service_version` で比べられる。

### B. 再試行後の成功

```text
INFO  event=export.accepted   request_id=req-8b21 job_id=job-1025 report_type=monthly
INFO  event=export.started    job_id=job-1025 queue_wait_ms=150
DEBUG event=renderer.attempt_completed job_id=job-1025 attempt=1 outcome=failure
      error_code=renderer_timeout duration_ms=3000
DEBUG event=renderer.attempt_completed job_id=job-1025 attempt=2 outcome=success duration_ms=700
INFO  event=export.completed  job_id=job-1025 outcome=success attempt_count=2
      recovered=true page_count=12 size_bytes=284000 duration_ms=3850
```

失敗した試行は一件あるが、ジョブは成功している。失敗件数を数えるときに、試行の失敗をジョブの失敗に足さない。`recovered=true` と `duration_ms` から、成功していても依存先の劣化で遅くなったことは追える。

### C. 最終失敗

```text
INFO  event=export.accepted   request_id=req-c40d job_id=job-1026 report_type=monthly
INFO  event=export.started    job_id=job-1026 queue_wait_ms=130
DEBUG event=renderer.attempt_completed job_id=job-1026 attempt=1 outcome=failure
      error_code=renderer_timeout duration_ms=3000
DEBUG event=renderer.attempt_completed job_id=job-1026 attempt=2 outcome=failure
      error_code=renderer_timeout duration_ms=3000
DEBUG event=renderer.attempt_completed job_id=job-1026 attempt=3 outcome=failure
      error_code=renderer_timeout duration_ms=3000
ERROR event=export.completed  job_id=job-1026 outcome=failure attempt_count=3
      error_code=renderer_timeout failed_step=render duration_ms=9300
```

ERRORはジョブとして期待した成果を出せなかった一件だけに付ける。各試行のタイムアウトは同じ原因の詳細なので、ERRORを重ねない。

### D. 途中停止

```text
INFO  event=export.accepted   request_id=req-d913 job_id=job-1027 report_type=monthly
INFO  event=export.started    job_id=job-1027 queue_wait_ms=110
DEBUG event=renderer.attempt_completed job_id=job-1027 attempt=1 outcome=success duration_ms=650
（このあと何も見つからない）
```

PDFの生成までは進んだ。そこから先は、保存中・コミット中にワーカーが停止した、完了記録が収集経路で失われた、検索条件が合っていない、のどれかである。ログだけでは決められないので、DBのジョブ状態とストレージ上のファイルの有無を確かめる。「受付から10分経っても `done` にならないジョブの数」をメトリクスで見ておけば、ログが欠けていても停滞に気付ける。

## レベル・処理結果・通知の判断表（一例）

この例での選び方である。同じ判断を全システムに当てはめる規則ではないが、自分の処理で表を作るときの型になる。

| 場面 | 記録する出来事 | レベル | 処理結果（`outcome`） | 通知 |
| --- | --- | --- | --- | --- |
| 未対応のレポート種別で受付を拒否 | `export.rejected` | INFO | `rejected` | しない。拒否率の急増はメトリクスで見る |
| 試行が失敗し、次の試行がある | `renderer.attempt_completed` | DEBUG | `failure`（試行の結果） | しない |
| 再試行後に成功（場面B） | `export.completed` | INFO | `success`、`recovered=true` | しない。再試行率や所要時間の悪化はメトリクスで見る |
| グラフ描画部分が使えず、グラフを省いた簡易版PDFを作った | `export.completed` | WARN | `success`、`degraded=true` | しない。劣化した応答の割合が目標を外れたら通知 |
| 再試行を使い切って失敗（場面C） | `export.completed` | ERROR | `failure` | 一件では呼ばない。失敗率が目標を外れたら通知 |
| PDFは保存したがDBの状態が更新できず、ファイルと状態が食い違った | `export.inconsistent` | ERROR | `failure` | 一件でも対応を求める。利用者が取得できず、自動では直らないため |
| 完了記録が来ない（場面D） | 記録なし | — | 不明 | ログではなく「一定時間未完了のジョブ数」のメトリクスで通知 |

読み方の要点は三つある。

- **レベル**は、記録した層から見た出来事の深刻さで選ぶ。試行の失敗はワーカーが自分で回復する前提なのでDEBUG、ジョブとして成果を出せなかったときだけERRORにする
- **処理結果**は、どの操作の結果かを決めてから書く。試行の `failure` とジョブの `failure` は別の数である
- **通知**は、ERRORの有無ではなく、利用者への影響と人が動く必要があるかで決める。率で見る失敗と、一件でも対応が要る食い違いを分ける

各場面の根拠と考え方は、[イベントの設計](event-design.md)、[処理をつなぐID](correlation.md)、[レベル・結果・通知](severity-and-outcomes.md)、[記録の届き方と限界](reliability.md)で説明する。
