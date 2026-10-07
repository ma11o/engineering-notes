# ログレベル・処理結果・通知を分ける根拠

確認日: 2026-10-07。公開一次資料のみを調査した。以下は書籍本文の引用ではなく、[書籍ノート](../reading-notes.md)第6節の仮のレベル表を検証するための外部根拠と設計上の解釈である。

## 要点

仮のレベル表には「どの操作・どの層での出来事か」を加えるとよい。例外、操作の失敗、ERRORログ、人への通知は一対一に対応しない。下位の呼び出しが失敗しても上位の処理が復旧することがあり、逆に正常なコード経路でも要求した成果や期限を満たさなければ利用者には失敗となる。後半は本ノートの設計判断であり、特定製品が保証する動作ではない。

## 採用資料と主張の対応

### S1: レベル名の共通語彙を支える仕様

- 著者・発行主体: OpenTelemetry Authors / OpenTelemetry
- 題名: Logs Data Model
- URL: https://opentelemetry.io/docs/specs/otel/logs/data-model/#field-severitynumber
- 補助箇所: https://opentelemetry.io/docs/specs/otel/logs/data-model/#error-semantics
- 閲覧日: 2026-10-07。ページ表示は OTel 1.61.0、Status: Stable。公開日・最終更新日は本文から確定していない
- 種別: OTelデータモデルの規範。個別アプリの運用規則ではない

根拠: DEBUGはデバッグ用の出来事、INFOは出来事の発生を知らせる情報、WARNはエラーではない警告、ERRORは誤りが生じた出来事と区分する。範囲は順に5–8、9–12、13–16、17–20。ERROR以上の値を受信後どう利用するかは読み手の判断に委ねる。

解釈・適用: 仮のレベル表のレベル名には外部の足場がある。一方、「WARNなら再試行」「ERRORなら即時通知」という対応はこの仕様からは導けない。既存ロガーをOTelへ変換する場合も、名前だけでなく元のレベルの意味を確認する。

### S2: 失敗を測る操作境界と正常母集団

- 著者・発行主体: OpenTelemetry Authors / OpenTelemetry
- 題名: Recording errors
- URL: https://opentelemetry.io/docs/specs/semconv/general/recording-errors/#what-constitutes-an-error
- 補助箇所: https://opentelemetry.io/docs/specs/semconv/general/recording-errors/#recording-errors-on-metrics
- 閲覧日: 2026-10-07。ページ表示は Semantic conventions 1.44.0、Status: Development。公開日・最終更新日は未確定
- 種別: 計装・semantic conventions著者向けの発展中の推奨。全ログの出力禁止規則ではない

根拠: 404も、存在を期待した取得と存在確認では失敗判定が異なる。再試行・処理によって完了した操作のspanやmetricには、その途中のエラーを操作の失敗として記録しないことを推奨する。成功時にはerror.typeを付けず、成功・失敗を含む一つのmetricを推奨する。

解釈・適用: 「よく起こる失敗」と「操作として成功」は別である。復旧できたかは観測対象の操作単位で判定する。途中の試行や復旧経路を調べるログまで一律に削除する根拠にはしない。正常比較用の分母は失敗ログ件数から作らず、対象操作全体を数える。

### S3: WARNログでも、その層の操作は失敗しうる

- 著者・発行主体: OpenTelemetry Authors / OpenTelemetry
- 題名: Semantic conventions for exceptions in logs
- URL: https://opentelemetry.io/docs/specs/semconv/exceptions/exceptions-logs/#severity
- 閲覧日: 2026-10-07。Semantic conventions 1.44.0。ページ全体は「Stable, except where otherwise specified」だが、ここで利用するSeverity節はDevelopment。公開日・最終更新日は未確定
- 種別: 例外イベントの発展中の規範的推奨。全アプリケーションログへの機械的な適用はしない

根拠: Severityは例外の有無より予想される影響を表す。アプリが処理すると期待する例外をWARN、アプリが処理せず終了には至らない例外をERRORとすることを推奨する（SHOULD）。例として、クライアントライブラリが再試行を使い果たした後の失敗を一件のWARNにし、個々の再試行記録は下位計装へ任せる。

解釈・適用: 「再試行枯渇＝常にERROR」「WARN＝必ず自動復旧済み」は一般則にできない。ライブラリから呼出元へ失敗を伝える層と、利用者への最終結果を確定する層を区別する。独自のレベル表には適用層と処理責任を書く。

### S4: 人への通知は利用者影響と分母で設計する

- 著者: Steven Thurgood、協力 Jess Frame, Anthony Lenton, Carmela Quinito, Anton Tolchanov, Nejc Trdin
- 題名: Alerting on SLOs（The Site Reliability Workbook 第5章）
- 発行: Google / O’Reilly Media。ページの著作権表示は2018年
- URL: https://sre.google/workbook/alerting-on-slos/
- 参照節: Alerting Considerations、Ways to Alert on Significant Events、Low-Traffic Services and Error Budget Alerting
- 閲覧日: 2026-10-07
- 種別: Google SREの運用設計の知見。Prometheusの例示を含むが普遍的な通知閾値の規格ではない

根拠: 利用者が経験する信頼性をSLOで捉え、重要なerror budget消費を通知する。良い／悪いイベントを全イベントに対する比率で測る。通知設計は適合率、再現率、検知時間、解消後の通知継続時間で評価する。低トラフィックでは一件の失敗でも率が大きく動き、高価値な一件と一過性の失敗では扱いが異なる。

解釈・適用: ERRORの存在だけをページング条件とせず、影響、時間窓、件数、分母を含む別のルールを持つ。ログはその通知後の説明材料にもなる。SLO方式だけでセキュリティ・監査・データ破損の検知まで充足するとは仮定しない。

## 仮のレベル表に加える設計判断

以下は上記資料を踏まえた本ノートの提案であり、OTelの必須フィールドや組織標準ではない。

| 観点 | 決めておくこと | 誤読を避ける例 |
|---|---|---|
| 操作の成果 | 何が成立すれば成功か、期限も成功条件か | 存在確認の「なし」と、必須データ取得失敗を区別する |
| 観測境界 | 一回の試行、ライブラリ操作、業務処理のどれか | 下位のtimeoutと上位の最終結果を同じ分母に混ぜない |
| 復旧 | 復旧手段、試行回数、最終成果 | 自動復旧に成功しても所要時間悪化を別に観測する |
| 記録の意図 | 調査、通常成果の比較、異常原因、監査のどれか | DEBUG/INFOを正常・異常の二値ラベルとして使わない |
| 通知 | 誰が、何を、いつまでに変える必要があるか | 瞬間的な一件と、持続する利用者影響を分ける |

### 正常な比較対象を残すためのチェック

1. 業務処理を分母にするなら、一処理の最終結果と各retryを別集計にする。重複記録で「失敗率」を水増ししない
2. 成功／失敗の両方について、同じ定義のoperation、時間帯、リリース、経路などを比較できるようにする。これらの具体的な分割軸は本ノートの仮案
3. サンプリング・欠落のあるログから率を計算するなら、その偏りを明示する。全量metricと抽出ログを同じ母集団だと仮定しない
4. 「復旧したから問題なし」と閉じず、遅延や復旧回数の増加を確認する。ただし集団を細分化しすぎると分母が小さくなるため、件数も併記する

## 採用しなかった候補

OpenTelemetry Authors / OpenTelemetry「Logs」
URL: https://opentelemetry.io/docs/concepts/signals/logs/#semistructured-logs
閲覧日: 2026-10-07。ページの最終更新表示は2026-08-07

半構造化ログの説明例には、パスワード不一致のログをERRORにした例がある。しかし例の目的はログ形式の説明であり、「すべての認証不一致をERRORにすべき」という分類根拠にはならないため、レベル基準の根拠としては不採用とした。公式資料内でも、例示と規範を区別する必要がある。

## 適用前提・未検証事項

- 一般的なサーバー／ライブラリの観測設計を想定する。規制上の監査、セキュリティ事象、金銭・安全に関わる一件単位の重大性は別途設計が必要
- 外部根拠は仮のレベル表の各例を正当化しきらない。採用するライブラリ、計装バージョン、例外の所有層、SLO、通知運用に照らして検証する
- 実際のSDK・サービスの出力や設定は検証していない。仕様の記載と製品の実挙動を同一視しない
- Development節は変更されうる。チームで固定する際は対象バージョンを明記し、更新時に差分を確認する
