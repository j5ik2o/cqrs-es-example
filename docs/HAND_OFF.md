# cqrs-es-example 群 引継ぎ（HAND_OFF）

最終更新: **2026-11-08 00:58 JST**
更新担当: cqrs-es coordinator（Orca）
この文書は、担当者がコンテキストやレート制限で交代しても、次の担当がこの 1 枚で引き継げるようにするためのものです。実装の詳細は各 repo の issue / PR と `docs/acceptance-criteria.md` を参照してください。

## 0. 目的と対象

- 対象: `cqrs-es-example`（ハブ）と 6 つの言語別 repo（go / java / js / rs / scala / kotlin）。
- 目標: 全言語で ESA v5（go / java / kotlin / scala は 5.1.0、js / rs は 5.2.0）、DynamoDB 3 テーブル、envelope API、共通 GraphQL（GroupChat）、Write API / Read API / Read Model Updater（RMU）、docker-compose + LocalStack の e2e、main CI 緑。
- 既存データの移行は不要（サンプル）。

## 1. いまの状態（サマリ）

- **M0 受け入れ基準**: 完了（ハブ PR #6、設計判断 a〜k 確定）。
- **M1 赤 CI 修正**: 完了（go #673、java #589 をマージし、main CI 緑）。
- **M2 ESA v5 移行**: 移行本体は go / java / js / rs で main に反映済み。ただし **M2 はどの言語もまだ完了ではない**（復旧が残る）。go は #675〜#679 完了だが **#680 復旧が作業中**。java は #618〜#622 完了で **#623 復旧の PR #624 がレビュー対応中**。js は #831〜#835 完了で **#836 復旧の PR #837 がレビュー対応中**。rs は #598（RMU）・#599（Write API）・#601（Read API）完了で、**#550 e2e の PR #602 が CI 中**、#600 復旧が未着手。scala は M4 として移行完了だが、**DY-15 の再同期（head Scan）が未実装**で、復旧の追跡先は未定（決まり次第ここに追記する）。
- **M2 復旧（A 案）**: 2026-11-07 23:37 加藤さん決定。M2 を「head Streams 移行」と「復旧（再試行・DLQ・再同期）」に分離し、復旧を言語ごとの issue で実装する。追跡はハブ #13。go #680 / java #623 / js #836 / rs #600。
- **Renovate は停止中（2026-11-07 19:33 加藤さん決定）**: 使用量削減のため Renovate は off。既存 Renovate PR の修正・リベース・整理・クローズは禁止（WIP 超過解消目的でも）。作業者を Renovate PR に着手させない。マージは Grok Bot が CI 全緑＋未対応コメントなしのものだけ行う。WIP 超過が Renovate だけなら新規着手せず待機。**この停止中の方針は acceptance-criteria §10.h（patch 自動マージ、minor/major 手動）より優先する。**
- **M3 Kotlin**: 未着手。**java の v5 版が骨格**のため、java M2（復旧を含む）の完了待ち。
- **M4 Scala**: **完了**（scala #5 / #6 / #7 をマージ、issue #2 / #3 / #4 とマイルストーンをクローズ）。

## 2. マイルストーンと完了条件

### M2 の追加完了条件（2026-11-07 15:33 加藤さん決定・重要）

ESA v5 への依存更新 PR をマージしただけでは M2 完了としない。各言語（rs / go / java / js、および M4 の scala）で次をすべて満たす。

1. RMU の購読先を **journal の Streams から集約 head の Streams** に切り替える。**INSERT と MODIFY の両方**で `head.events` を読み、`__config__` は除外する。journal の Streams は集約ごとの順序を保証しないため使わない。
2. 集約ごとの処理位置（`read_model_positions`）とリードモデルの更新を、**同じ DB トランザクション**で行う。
3. 古い seq と重複した seq は捨てる。seq が飛んでいたら journal から補う。
4. setup、Terraform、ローカルの reader も **3 テーブル構成**（journal / snapshot / head）にし、head を購読するようにそろえる。
5. 重複、逆順、欠番、バッチ途中の失敗、再起動のテストを用意して通す。

- 根拠: event-store-adapter `docs/spec/storage/dynamodb.ja.md`（DY-15）、ADR-0002、ハブ `docs/acceptance-criteria.md` L53-59。
- 移行 issue: rs #549 / go #669 / java #586 / js #828（すべて完了）。scala は #3 として実装済み（PR #7）。

### M2 復旧の確定仕様（A 案・2026-11-07 23:37 加藤さん決定）

1. **再試行**: 回数上限と最大経過時間を設け、超えたら指数バックオフで退避する。Lambda は `BisectBatchOnFunctionError` を有効化し、必要なら `ReportBatchItemFailures` も使う。ローカル reader は指数バックオフ + ジッタ。
2. **永続復旧タスクと DLQ**: 失敗時は、ストア識別子・投影名・版・集約 ID・seq_nr・失敗種別を含む**永続復旧タスク**を記録する（これが復旧の正）。SQS / DLQ は配送専用であり、メッセージ保持期限に依存しない。復旧タスクは journal から該当イベントを読み直して冪等に再適用する。
3. **再同期（DY-15）**: 24 時間を超えて止まった場合や `TrimmedDataAccessException` では、**先に Streams の購読を再開**し、並行して head を ConsistentRead で全ページ Scan する。Scan 中の更新は journal から補完し、各集約の `head.seq_nr` と `read_model_positions` の差を冪等に適用する。`ExpiredIteratorException` はシャードごとに再開する。起動時も同じ手順で全件 Scan を行う。
4. **トランザクション**: 1 イベント、または上限つき範囲を 1 トランザクションで処理する。位置行とリードモデル更新は同一 TX。適用は「現在位置 + 1」のみ。ロック / 条件更新を使い、journal に欠番があれば位置を進めずに失敗させる。
5. **冪等性**: 同じイベントを何度適用しても結果が変わらないこと。
6. **テスト**: 必須 7 本（重複破棄、逆順破棄、欠番の journal 補完、バッチ途中失敗からの再試行、再起動後の再開、リトライ上限超過 → 永続復旧タスク → 復旧、24h 超え全再同期 / TrimmedDataAccess / ExpiredIterator）に加え、ハブ #13 が要求する追加テスト（同時適用、位置行の同時作成、複数ページ Scan、退避先への書き込み失敗）も M2 完了条件に含める。7 本だけでは完了としない。

### M3 Kotlin

- java の M2（v5 版の command / query / RMU と復旧）が完了してから着手する。
- kotlin #1（骨格 + command）→ #2（query/Read API + RMU）→ #3（compose + e2e + CI）の順。

### M4 Scala（完了）

- scala #2（ドメイン + Write API）→ #3（Read API + RMU、head Streams 条件を含む）→ #4（compose + LocalStack e2e + CI）を PR #5 / #6 / #7 でマージ済み。

## 3. repo ごとの open PR と issue

### ハブ `j5ik2o/cqrs-es-example`

- open PR: この HAND_OFF PR のみ（前回 #12 は `7d19cce` でマージ済み）。
- open issue:
  - #3 [M2] 追跡: ESA v5 移行（rs → js → java → go） https://github.com/j5ik2o/cqrs-es-example/issues/3
  - #4 [M3] 追跡: Kotlin 版の新規実装 https://github.com/j5ik2o/cqrs-es-example/issues/4
  - #13 [M2] 追跡: RMU 復旧（再試行・DLQ・再同期） https://github.com/j5ik2o/cqrs-es-example/issues/13
- マイルストーン: M2 / M3 が open。M0 / M1 / M4 は closed。

### go `j5ik2o/cqrs-es-example-go`

- open PR: 0（Renovate も 0）
- open issue:
  - #680 [M2] RMU の復旧（再試行・DLQ・再同期） https://github.com/j5ik2o/cqrs-es-example-go/issues/680 — **作業中**
  - #672 [M1 後続] staticcheck の既存指摘 41 件 https://github.com/j5ik2o/cqrs-es-example-go/issues/672
- 補足: #666〜#671 は完了・クローズ済み。PR #675〜#679 と #573 はマージ済み。M2 移行は完了、復旧 #680 が残る。

### java `j5ik2o/cqrs-es-example-java`

- open PR（Renovate 3 本のみ。停止方針により触らない）:
  - #616 Spring Boot v4 https://github.com/j5ik2o/cqrs-es-example-java/pull/616
  - #608 java-jdk v25 https://github.com/j5ik2o/cqrs-es-example-java/pull/608
  - #559 Gradle 9（close 候補） https://github.com/j5ik2o/cqrs-es-example-java/pull/559
- マージ済み（M2）: #618（ESA v5.1.0 / 3 テーブル）、#619（Write API）、#620（Read API）、#621（RMU head Streams）、#622（e2e）。issue #583〜#587 はクローズ済み。
- open issue:
  - #623 [M2] RMU の復旧（再試行・DLQ・再同期） https://github.com/j5ik2o/cqrs-es-example-java/issues/623 — **PR #624 が CodeRabbit 未解決 18 本を対応中**
  - #588 [M2] Renovate PR を整理 https://github.com/j5ik2o/cqrs-es-example-java/issues/588

### js `j5ik2o/cqrs-es-example-js`

- open PR（Renovate 3 本のみ。停止方針により触らない）:
  - #819 functions-framework v5 https://github.com/j5ik2o/cqrs-es-example-js/pull/819
  - #812 @types/node https://github.com/j5ik2o/cqrs-es-example-js/pull/812
  - #811 yargs v18 https://github.com/j5ik2o/cqrs-es-example-js/pull/811
- マージ済み（M2）: #831（ESA v5.2.0）、#832（Write API）、#833（Read API）、#834（RMU head Streams）、#835（e2e）。issue #825〜#829 はクローズ済み（#829 は加藤さん承認のうえクローズ）。
- open issue:
  - #836 [M2] RMU の復旧（再試行・DLQ・再同期） https://github.com/j5ik2o/cqrs-es-example-js/issues/836 — **PR #837 が CodeRabbit 未解決 2 本を対応中**
  - #830 [M2] Renovate 整理 https://github.com/j5ik2o/cqrs-es-example-js/issues/830

### rs `j5ik2o/cqrs-es-example-rs`

- open PR（Renovate 5 本のみ。停止方針により触らない）:
  - #597 Terraform aws v6 https://github.com/j5ik2o/cqrs-es-example-rs/pull/597
  - #592 aws_lambda_events v1 https://github.com/j5ik2o/cqrs-es-example-rs/pull/592
  - #591 mysql v26 https://github.com/j5ik2o/cqrs-es-example-rs/pull/591
  - #590 Helm mysql v14 https://github.com/j5ik2o/cqrs-es-example-rs/pull/590
  - #582 sqlx 0.9 https://github.com/j5ik2o/cqrs-es-example-rs/pull/582
- マージ済み（M2）: #598（RMU head Streams）、#599（Write API）、#601（Read API、`f958c548`）。
- open issue:
  - #550 [M2] compose + e2e https://github.com/j5ik2o/cqrs-es-example-rs/issues/550 — **PR #602 が CI 中**
  - #600 [M2] RMU の復旧（再試行・DLQ・再同期） https://github.com/j5ik2o/cqrs-es-example-rs/issues/600 — **未着手（#550 の後）**
  - #551 [M2] Renovate 整理 https://github.com/j5ik2o/cqrs-es-example-rs/issues/551

### scala `j5ik2o/cqrs-es-example-scala`

- open PR: 0
- open issue: 0
- M4 マイルストーン: closed。

### kotlin `j5ik2o/cqrs-es-example-kotlin`

- open PR: 0
- open issue:
  - #1 [M3] 骨格と command 側 https://github.com/j5ik2o/cqrs-es-example-kotlin/issues/1
  - #2 [M3] query/Read API と RMU https://github.com/j5ik2o/cqrs-es-example-kotlin/issues/2
  - #3 [M3] compose + e2e + CI https://github.com/j5ik2o/cqrs-es-example-kotlin/issues/3
- java M2 完了待ち。

## 4. 作業中の Orca 端末・worktree・担当 issue

- **go #680（復旧）**: worktree `m2-go-recovery` / terminal `term_f842fb44-66a7-4f09-9628-f31b54c751ba` / task `task_e2b6649475ed`
- **java #623（復旧）**: worktree `m2-java-recovery` / terminal `term_359fb8e4-00e6-41b4-92bf-a1ad293d04fa` / task `task_ec89d8678aa7`
- **js #836（復旧）**: worktree `m2-js-recovery` / terminal `term_137f680c-ddfb-456a-affe-617734fe1248` / task `task_6f6b36314f16`
- **rs #548（Read API）**: worktree `m2-rs-read-api` / terminal `term_4483f248-12b4-4d4f-9b3c-aaa2e8383487` / task `task_b9f5371bba6b`
- 完了済み worker の端末は指示により close せず orphaned のまま: `term_18bfc47d`（go #667）、`term_dd421d56`（go #668）、`term_7d24356a`（go #669）、`term_313c97bc`（go #679）、`term_ea435055`（go #671）、`term_df43c75b`（go #573）、`term_f28d70c0`（scala #7）、`term_21227747`（scala #3 検証）、`term_22ca3215`（java #614）、`term_20839d9c`（js #823）。
- 端末上限対応で idle シェルを閉じた: scala main `term_6aee73dd`、hub `term_4f1d95f0`。

## 5. 判断待ち

- **rs #591（mysql v26）**: Renovate PR。変更が `tools/docker-compose/docker-compose-databases.yml` のみで、ci.yml の `paths-ignore: tools/**` により CI が走らない。healthcheck のローカル修正（`mysqladmin ping`）と commit `20707fc` は保持するが、**Renovate 停止中は push・paths-ignore 変更・admin merge をしない**。進めるには加藤さんによる明示的な方針変更が必要。マージは Renovate 停止方針（Grok Bot が CI 全緑＋未対応コメントなしのみ）に従う。#592 の `local.rs` も未コミットのまま保持する。
- **close 候補（勝手に close しない。加藤さんの判断待ち）**
  - java #559（Gradle 9）
  - js #819 / #812 / #811
- 設計判断 a〜k（M0）は **2026-11-06 に加藤さんが推奨どおり確定**。`docs/acceptance-criteria.md` §10 は「判断待ち」として列挙しているが、内容は確定済みとして読む。確定の記録はハブ issue #1 のコメントを参照: https://github.com/j5ik2o/cqrs-es-example/issues/1
- 復旧の設計論点（リトライ上限、DLQ、24 時間超停止後の再同期）は **2026-11-07 23:37 に A 案で確定**。残るのは各言語での実装とテスト。

## 6. 加藤さんの決定・禁止事項

- **WIP は repo ごと**: 開いている PR は 1 repo 3 本まで（ドラフト含む）。作業者は 1 repo 1 本。repo が違えば並行可。**新規 PR を作る前に、その repo の open PR を 2 本以下にしておく**（3 本のまま新規 PR を出すと上限超過）。
- **Renovate は停止中（2026-11-07 19:33 加藤さん決定）**: 使用量削減のため Renovate は off。既存 Renovate PR の修正・リベース・整理・クローズは禁止（WIP 超過解消目的でも）。作業者を Renovate PR に着手させない。マージは Grok Bot が CI 全緑＋未対応コメントなしのものだけ行う。WIP 超過が Renovate だけなら新規着手せず待機する。**この停止中の方針は acceptance-criteria §10.h（patch 自動マージ、minor/major 手動）より優先する。**
- **Bugbot 確認必須**: Bugbot / CodeRabbit の未解決スレ・未対応コメントがある PR はマージしない。Codex の usage limit と CodeRabbit の rate-limit のコメントは指摘扱いしない。
- **M2 完了条件**: 上記 2 章の head Streams 条件と復旧 A 案を満たすこと。依存更新 PR のマージだけでは完了にしない。
- **禁止**:
  - AGENTS.md の編集
  - docker prune
  - 他担当のコンテナ・画像・worktree・terminal の削除
  - 公開・リリース
  - force push / rebase（同期は `git merge origin/main` または GitHub の Update branch を使う）
  - **PR #83 の再実装**（対象は j5ik2o/event-store-adapter（common）の https://github.com/j5ik2o/event-store-adapter/pull/83 のみ。go / java / js / rs の #83 ではない。**拒否済み・再開しない**）
  - ESA 本体 repo（event-store-adapter-*）の変更
- **main への直 push 禁止**（PR 経由）。ただし HAND_OFF は下記の運用に従う。
- **HAND_OFF の運用**:
  - main へ直 push せず、ブランチから PR を作る。PR タイトル・本文は日本語、コミットは英語 Conventional Commit（例: `docs: update hand-off`）。
  - CI が緑で未対応コメントがなければ squash でマージ。
  - HAND_OFF の PR は同時に 1 本まで。前の PR がマージされるまで次を作らない。
  - 更新は、マイルストーンの完了や大きな方針変更のたび、およびそれとは別に最長 1 時間おき。
  - マージ後は worktree をローカルで破棄（リモート枝は消さない）。
- **close 候補は close しない**（加藤さんの判断待ち）。

## 7. 次にやること

1. **go**: 復旧 #680 の PR #681 の CI を確認し、緑・未解決 0 でマージ待ちにする（worker 稼働中）。
2. **java**: 復旧 #623 の PR #624 で CodeRabbit 未解決 18 本を修正または理由つき resolve させる（worker 対応中）。完了後、java M2 が完了したら kotlin M3 の骨格に着手できる。
3. **js**: 復旧 #836 の PR #837 で CodeRabbit 未解決 2 本を修正または理由つき resolve させる（worker 対応中）。
4. **rs**: #550 e2e の PR #602 の CI を確認してマージ待ちにし、その後 #600 復旧に着手する。
5. **kotlin M3**: java M2（復旧を含む）完了後に #1 → #2 → #3。
6. **HAND_OFF**: 最長 1 時間おき、およびマイルストーン完了・方針変更時に更新。
7. **Renovate**: 停止中。既存 PR は触らず、Grok Bot のマージ判断のみ待つ。

## 8. 参照

- 計画: `/Users/j5ik2o/orca/handoff/cqrs-es/cqrs-es-plan.md`
- 受け入れ基準: https://github.com/j5ik2o/cqrs-es-example/blob/main/docs/acceptance-criteria.md
- ESA v5 DynamoDB 仕様: https://github.com/j5ik2o/event-store-adapter/blob/main/docs/spec/storage/dynamodb.ja.md
- ADR-0002: https://github.com/j5ik2o/event-store-adapter/blob/main/docs/adr/0002-separate-aggregate-head-from-snapshot.md

## 更新履歴

- 2026-11-07 16:10 JST: 初版。M0/M1/M4 完了、M2 進行中、M3 待ちの状態を記録。
- 2026-11-07 16:15 JST: CodeRabbit 指摘（再同期の既定手順と未実装範囲、a〜k の確定参照、PR #83 の対象、新規 PR 用の WIP 枠）を反映。
- 2026-11-07 17:37 JST: go #668/#669 完了、scala #3 の再確認・クローズ、PR #83 の対象を common repo に確定、open PR 一覧を更新。
- 2026-11-07 18:38 JST: go #679（e2e）再 CI 待ち、java #614 / js #823 / rs #595・#596 のマージを反映。WIP を go 2 / java 4 / js 4 / rs 5 に更新し、rs #591 の paths-ignore 判断待ちを明記。
- 2026-11-07 18:52 JST: go #679 マージ（`4fb791cd`）と #670 クローズを反映。M2 go の残りは #671（Renovate 整理、トリアージ中）。WIP は go 1 / java 4 / js 4 / rs 5。
- 2026-11-07 19:20 JST: go #573（mysql v26）マージ（`00067bda`）と #671 クローズ、go M2 マイルストーン closed を反映。go の M2 完了（open PR 0）。
- 2026-11-07 19:46 JST: Renovate off（19:33 加藤さん決定）を反映。java #400 / js #815 のマージ、WIP go 0 / java 3 / js 3 / rs 5、Renovate PR の修正・整理・close 禁止と Grok Bot マージ方針を明記。
- 2026-11-07 19:54 JST: CodeRabbit 指摘（停止方針が acceptance-criteria §10.h より優先すること、rs #591 の旧手順を削除して明示的な方針変更が必要とすること）を反映。
- 2026-11-07 23:58 JST: java #618〜#622、js #831〜#835、rs #598・#599 のマージを反映。java #587 / js #829 クローズ、M2 復旧 A 案（hub #13、go #680 / java #623 / js #836 / rs #600）と 4 人の作業者を記録。rs #548 Read API を進行中に更新。
- 2026-11-08 00:58 JST: rs #601（Read API）マージ（`f958c548`）と #602（e2e）CI 中を反映。M2 は復旧まで未完了であること、scala の DY-15 再同期が未実装であることを明記。DLQ を配送専用、復旧タスクを永続記録として役割分担し、再同期は購読再開→並行 Scan→journal 補完の順に修正。追加テスト（同時適用、位置行の同時作成、複数ページ Scan、退避先への書き込み失敗）を完了条件に追加。
