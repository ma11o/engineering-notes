# ログのイベント設計と相関 ID：公式資料の主張台帳

確認日：2026-10-07（UTC）。一般公開の公式仕様のみを直接確認。個別システムの実装・適合性は検証していない。

## 結論

既存の「構造化ログ・イベント名・相関 ID」の説明に足す価値があるのは、**モデルと意味規約の適用範囲、ID の任意性、伝播時の信頼境界**である。以下の「規範」は該当仕様への適合条件であり、すべてのアプリケーションに一律適用される義務ではない。

## S1：一般ログモデルの許容範囲

- 資料名：Logs Data Model
- 著者・責任主体：OpenTelemetry Authors／OpenTelemetry Specification contributors（個人著者の記載なし）
- URL：https://opentelemetry.io/docs/specs/otel/logs/data-model/
- 参照箇所：Events、Trace Context Fields、Body、Attributes、EventName
- 閲覧日：2026-10-07
- 確認できた状態：Stable。取得時のサイト表示は OTel 1.61.0

**主張［仕様上の定義・規範］**

1. `EventName` はイベントの型を識別し、属性と本文の構造を一意に示すことが SHOULD。空でないイベント名を持つ LogRecord が Event に当たる。ただし、すべての LogRecord を Event にする義務はない。
2. 一般の `Body` は任意で、文字列に限らず配列・マップ等の構造化値を許す。モデルは AnyValue を扱える必要がある（MUST）。`Attributes` は個々の発生に付随する追加情報である。
3. `TraceId`・`SpanId` はともに任意。`SpanId` がある場合は `TraceId` もあることが SHOULD。

**限界**：欄名を揃えるだけでは SDK、転送形式、バックエンドを含めた OTel 適合を示せない。任意欄を全ログに必須とする設計は、採用側の追加ルールである。

## S2：名前付きイベントを設計する場合の追加ルール

- 資料名：Semantic conventions for events
- 著者・責任主体：OpenTelemetry Authors／Semantic Conventions contributors（個人著者の記載なし）
- URL：https://opentelemetry.io/docs/specs/semconv/general/events/
- 参照箇所：When to define events、Event name、Attributes、Body
- 閲覧日：2026-10-07
- 確認できた状態：Development。取得時のサイト表示は Semantic conventions 1.44.0

**主張［意味規約内の規範・設計ガイダンス］**

1. イベント名は構造を識別する必要があり（MUST）、発生ごとに変わる値を含めてはならない（MUST NOT）。識別子等の可変値は属性に置く。
2. 意味規約は構造化された詳細・文脈を属性として定義する。`Body` の値の定義は表示用文字列の場合に限られる。これは一般 LogRecord の構造化 Body を禁止する話ではない。
3. 独立した時点の発生・状態変化等には Event、時間幅と境界を持つ操作には Span、操作全体の性質には Span 属性を使い分ける。Event はアクティブなトレースの外でも発生できる。

**限界**：Development の規約を「確定済みの普遍ルール」と書かない。ログ一般のモデル、イベント意味規約、導入側の命名規則を混同しない。

## S3：既存 JSON ログへの橋渡し

- 資料名：Trace Context in non-OTLP Log Formats
- 著者・責任主体：OpenTelemetry Authors／OpenTelemetry Specification contributors（個人著者の記載なし）
- URL：https://opentelemetry.io/docs/specs/otel/compatibility/logging_trace_context/
- 参照箇所：Overview、JSON Formats
- 閲覧日：2026-10-07
- 確認できた状態：Stable

**主張［互換性規約］**

非 OTLP ログでは `trace_id`・`span_id` を小文字の十六進表記で記録する名前と表現が示される。`trace_flags` は W3C の形式に従う。3 欄は任意で、JSON ではトップレベルに置くことが SHOULD。

**限界**：この推奨は非 OTLP 形式の橋渡しであり、OTLP 全体の JSON 表現を定義するものではない。`request_id` の文字列を単に `trace_id` に改名してよい根拠にもならない。例の ID を実装時の長さ検証用のテスト値として流用しない。

## S4：伝播と信頼境界

- 資料名：Trace Context
- 著者・責任主体：W3C Distributed Tracing Working Group
- 文書記載の編集者：Sergey Kanzhelev、Morgan McLean、Alois Reitbauer、Bogdan Drutu、Nik Molnar、Yuri Shkuro
- 検証・固定版 URL：https://www.w3.org/TR/2021/REC-trace-context-1-20211123/
- 参照箇所：§3.2.2、§3.4、§6、§7。§4 の処理例は non-normative
- 閲覧日：2026-10-07
- 確認できた状態：W3C Recommendation、2021-11-23

**主張［規範／セキュリティ上の説明］**

1. HTTP の `traceparent` はトレース全体の `trace-id` と呼び出し側の操作を示す `parent-id` 等を伝える。v00 では前者は 16 byte、後者は 8 byte の小文字十六進表現で、全ゼロは無効。受信 `parent-id` を受信側の現在 Span ID と同一視しない。
2. `traceparent`・`tracestate` を個人識別情報や機微情報に使ってはならない（MUST NOT）。セキュアなネットワーク入口ではトレース再開始という変更方法が認められる。
3. 外部入力のヘッダーを無条件に信用すると、相関データの汚染や過剰な収集負荷・費用につながりうる。`sampled` は保存保証ではない。ヘッダーの長さ・内容検査等が必要になる。

**限界**：本資料は HTTP 伝播の仕様であり、ロガーが実行コンテキストを自動継承することや、キューを含む全経路の追跡成功を保証しない。信頼境界で毎回必ず ID を変更せよ、という一律の要件ではない。

## S5：相関 ID にも領域固有の意味がある

- 資料名：Messaging（Semantic Conventions attribute registry）
- 著者・責任主体：OpenTelemetry Authors／Semantic Conventions contributors（個人著者の記載なし）
- URL：https://opentelemetry.io/docs/specs/semconv/registry/attributes/messaging/
- 参照箇所：General Messaging Attributes の `messaging.message.conversation_id` と `messaging.message.id`
- 閲覧日：2026-10-07
- 確認できた状態：両属性とも Development

**主張［領域固有の意味規約］**

`messaging.message.conversation_id` はメッセージが属する会話の ID で、Correlation ID と呼ばれる場合もある。一方、`messaging.message.id` はメッセージシステムがそのメッセージを識別する値。会話・メッセージ・トレースという異なる単位を、名前だけで同一視できない。

**限界**：メッセージング向けの語彙であり、HTTP や全アプリケーションの汎用 `correlation_id` の伝播・生成ルールを定めるものではない。

## 推論と採用側の提案（仕様の直接引用ではない）

- **限定した推論**：S1・S4 は汎用 `request_id`／`correlation_id` の共通欄と一律の寿命を規定していない。S5 のような領域固有規約は存在するため、「相関 ID には標準が一切ない」とも書かない。どの単位を束ねる ID かは、対象の仕様・実装契約で確かめる。
- **独自提案**：設計表には、ID ごとの生成者、対象単位、再試行時の扱い、保持期間、外部へ渡す境界を記す。例として「受信単位の request ID」「複数の試行を束ねる業務相関 ID」「トレース」「Span」を別々に説明する。ただし、この区分自体を普遍仕様としない。
- **独自提案**：ログには現在の実行コンテキストから取れる ID を付け、ないものを捏造しない。入口・HTTP 呼出し・非同期投入／実行・終了処理でコンテキストの継承と解放をテストする。外部入力の相関値を認証・認可判断に使わない。
- **独自提案**：OTel 採用の有無と別に、イベント名・属性・表示文の責務と変更方針を定める。導入確認前は「OTel に準拠している」ではなく、「公式モデルを参照した設計上の整理」とする。

## 本文に反映する際の優先順位

1. `EventName`／`Body`／`Attributes` の区別に S1 と S2 の適用範囲・成熟度を添える
2. trace/span ID を全ログ必須として扱わず、利用可能な実行コンテキストとの対応を説明する
3. request/correlation ID の寿命を採用側の契約として示し、受信値の信頼と外部伝播を別の設計課題として加える
4. JSON への `trace_id` 等の配置は S3 の互換性規約として扱い、実装適合の断定を避ける

## 採用しなかった候補と現行性の限界

- W3C Distributed Tracing Working Group「Trace Context Level 2」<https://www.w3.org/TR/trace-context-2/>（閲覧日2026-10-07）は取得時点でCandidate Recommendation Draftと表示された。今回の確定済み規範の根拠には2021年Recommendation固定版を採用した。Level 2の差分を網羅した調査ではなく、S4を「すべての最新仕様」とは称さない
- 一般的な「request IDとcorrelation IDの違い」の解説記事は追加しなかった。命名の慣例を普遍仕様へ誤って引き上げるより、対象領域の公式属性と採用側の契約を分ける方が、この監査の不足を直接補えるため
