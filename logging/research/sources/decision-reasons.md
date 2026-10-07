# 判断の理由を記録する先行例：機能フラグの評価理由

確認日: 2026-10-08。公式の仕様と製品ドキュメントだけを調べた。実装での動作確認はしていない。

[共通例2：予約の問い合わせ](../../example-reservation.md)では、予約可能日APIが「なぜその日を予約できないと判断したか」を理由の分類つきで残した。システムが判断した結果に理由を添えて記録する考え方は、機能フラグ（feature flag）の分野で仕様や製品の機能として整っている。機能フラグの評価とは、「この利用者にはこの機能を有効にするか」をルールに照らして決めることで、予約可能日の判定と同じく、入力・ルール・結果・理由がある。

## 資料と主張

### OpenFeature：評価の詳細と理由

- 資料: [OpenFeature Specification: Types](https://openfeature.dev/specification/types)
- 評価の詳細（Evaluation Details）は、フラグのキー、値、variant（値の名前）、理由（reason）、エラーコード、エラーメッセージ、フラグのメタデータを持つ
- 理由には定義済みの値がある。`STATIC`（動的な評価なし）、`DEFAULT`（既定値に戻った）、`TARGETING_MATCH`（ルールや利用者の指定に一致した）、`SPLIT`（擬似乱数による割り当て）、`CACHED`、`DISABLED`、`UNKNOWN`、`STALE`、`ERROR`。定義済みの値以外の任意の文字列も使える

### OpenTelemetry：評価イベントの意味規約

- 資料: [Semantic conventions for feature flags in events](https://opentelemetry.io/docs/specs/semconv/feature-flags/feature-flags-events/)
- 状態: 文書全体はDevelopment、`feature_flag.evaluation` イベントと属性の多くはRelease Candidate（2026-10-08確認）
- イベント名は `feature_flag.evaluation`。フラグの値を評価するたびに、結果が同じでも出すことが推奨（SHOULD）されている
- 属性は、`feature_flag.key`（必須）、`feature_flag.result.variant`・`feature_flag.result.value`（条件付き必須）、`feature_flag.result.reason`・`feature_flag.context.id`・`feature_flag.version`（推奨）など。`feature_flag.version` は「評価に使ったルールセットの版」とされている
- 値（`feature_flag.result.value`）は大きくなったり機微情報を含んだりし得るので、値そのものを含めずに参照できる variant が望ましいことが多い、と注記されている
- OpenFeatureの評価の詳細とOpenTelemetryの属性の対応は、[OpenFeature Specification: Appendix D: Observability](https://openfeature.dev/specification/appendix-d)にある

### LaunchDarkly：評価理由

- 資料: [LaunchDarkly: Evaluation reasons（概念）](https://launchdarkly.com/docs/sdk/concepts/evaluation-reasons)、[Flag evaluation reasons（SDKの機能）](https://launchdarkly.com/docs/sdk/features/evaluation-reasons)
- 理由の種類は `OFF`、`FALLTHROUGH`（どのルールにも一致せず既定のルールの値を返した）、`TARGET_MATCH`、`RULE_MATCH`、`PREREQUISITE_FAILED`、`ERROR`
- `RULE_MATCH` には、一致したルールの位置（`ruleIndex`）と、ルールの一意な識別子（`ruleId`）が付く。`ruleId` はルールの並び順を変えても変わらない
- 理由は、`VariationDetail` 系のメソッドで評価したときに得られ、そのときは分析用のイベントにも含まれる

## 予約可能日の判定への当てはめ

以下は上の資料を踏まえた本ノートの解釈である。

| 機能フラグの評価 | 予約可能日の判定（共通例2） | 当てはめて分かること |
| --- | --- | --- |
| 理由を定義済みの分類で返す（`TARGETING_MATCH` など）。任意の文字列も許す | `slot_full`・`closed_day` などの理由の分類 | 理由は自由文でなく、安定した分類にする。分類の一覧を設計時に決める |
| 一致したルールの `ruleId`。並び順を変えても変わらない | 「最初に当てはまったルールだけを書く」場合 | 理由の分類に加えて、どのルールで外れたかをルールの識別子で残すと、ルールの順番を変えても記録の意味が変わらない |
| `feature_flag.version`（評価に使ったルールセットの版） | `rule_version` | ルールの版を判断の根拠として残す考え方は共通している |
| 値そのものより variant を勧める（大きい・機微な値を避ける） | 予約可能日の一覧ではなく `available_count` を残す | 判断の結果は、問いに答えられる範囲で小さく、機微でない形にする |
| `feature_flag.context.id`（評価の対象。targeting key など） | `user_ref` | 評価の対象を、直接の個人情報でない識別子で示す |
| 評価のたびにイベントを出すことを推奨 | 記録する回数は量と費用で決める（検討点） | 仕様は毎回の記録を勧めるが、量が問題になる処理では、問い合わせに要る範囲から回数を決める |
| LaunchDarklyでは理由を求めたときだけイベントに含める | — | 理由の記録には量や処理の負担があり、必要な箇所で選んで残す設計もあり得る |

## 適用前提・未確認のこと

- 機能フラグの評価は、一つのフラグについて一つの値を返す。予約可能日の判定は、日付ごとに理由があるので、一回の判定で複数の理由をまとめて残す点が違う
- OpenTelemetryの意味規約はRelease Candidate／Developmentであり、名前や要求が変わり得る
- 各製品・SDKの実際の出力は確かめていない
