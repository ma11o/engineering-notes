# engineering-notes

技術を自分の言葉で説明し、設計やレビューで判断するための、開発全般の学習ノート。テーマごとに `docs/` の下にフォルダを分ける。MkDocs で GitHub Pages に公開している: https://ma11o.github.io/engineering-notes/

## 構成

- `docs/` — サイトの中身（`mkdocs.yml` の `docs_dir`）。`index.md` が全体のトップページで、テーマの一覧を置く
- `docs/logging/` — ログ設計。`README.md` がテーマの入口
  - 章: `principles.md` → `event-design.md` → `correlation.md` → `severity-and-outcomes.md` → `security-and-privacy.md` → `reliability.md` → `review-checklist.md`
  - 共通例: `example.md`（共通例1：レポート出力ジョブ）、`example-reservation.md`（共通例2：予約の問い合わせ）
  - `tools/` — 製品に依存する実践（New Relic と CloudWatch の役割分担など）。一般論の章とは分ける
  - `research/` — 根拠資料。書籍ノート（`reading-notes.md`）、一次資料の確認記録（`sources/`）
- `docs/architecture-metrics/` — アーキテクチャの計測。`README.md` が本全体の要約で、テーマの入口。`research/reading-notes.md` が問い別の書籍ノート（根拠資料）
- 新しいテーマは `docs/<テーマ名>/README.md` を入口にして作り、`docs/index.md` と `nav` に足す
- `mkdocs.yml` — サイト設定と目次（`nav`）
- `.github/workflows/pages.yml` — `main` への push でビルドして Pages に公開

## 書き方

- 日本語で書く。各章の冒頭に「要点」を3つ、言い切りの形で置く
- 打ち消しや留保を重ねず、「ではどうするか」で終える。二重否定（「〜ではないという意味ではない」）は肯定の形に直す
- 全章に共通する前提（例は説明用、仕様の成熟度を明記する、など）は各テーマの `README.md` にまとめ、各章で繰り返さない
- 抽象的な説明は、共通例の具体的な値で言い直す。新しい例を作るより、そのテーマの共通例を使う（ログ設計は共通例1・2）
- 一般的な専門用語（WAL、原子性、SLO、span、カーディナリティ、index label など）は言い換えずに残し、初出で括弧に一言説明を添える。言い換えるのは、ノート独自の硬い言い回しだけ
- 共通例・イベント名・フィールド名・判断表は本ノート独自の提案であり、書籍の事例ではないと明記する
- 書き終えたテーマの日本語を整えるときは、`.claude/skills/note-polish` スキルの手順で、候補一覧を見せてから直す

## 出典と事実確認

- 外部の仕様や製品の挙動を書くときは、公式ドキュメントで確認し、リンクと確認日を残す。仕様は成熟度（Stable／Development など）も書く
- 書籍と、印刷ページ p.n と PDF のページの対応（本文と照合済み）
  - ログ設計: *Observability Engineering*, 2nd Edition。PDF の n+36 ページ目（2026-10-07）
  - アーキテクチャの計測: *Software Architecture Metrics*（O'Reilly, 2022）。PDF の n+14 ページ目（2026-10-09）
- 書籍の要旨と、自分の解釈・補足を分けて書く。書籍にない内容を書籍の主張として書かない
- 確認していないこと（実環境での動作、費用など）は「未確認」として書く

## 公開してよい内容

- 公開リポジトリ・公開サイトなので、会社固有の情報（実際のフィールド名、社内の取り決め、契約内容など）は書かない
- 非公開リポジトリ（`ma11o/todo` など）へのリンクは載せない

## ビルドと確認

- ページを追加したら `mkdocs.yml` の `nav` にも追加する
- 手元での確認: `pip install "mkdocs>=1.6,<2" "mkdocs-material>=9.7,<10"` のあと `mkdocs build --strict`（リンク切れで失敗する）または `mkdocs serve`
- MkDocs 2.0 は互換性がないため 1.x に固定している
- 日本語見出しのアンカーは `pymdownx.slugs.slugify` で GitHub と同じ形にしている。章をまたぐ見出しリンクは `example.md#レベル処理結果通知の判断表一例` の形で書く。リンクはそのファイルからの相対パスで、別テーマへは `../logging/example.md` のように書く

## コミット

- コミットの作者は ma11o。Claude を作者・共同作者にしない
- `main` へ直接コミットしてよい。push すると公開サイトに反映される
