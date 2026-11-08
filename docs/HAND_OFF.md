# cqrs-es-example 群 引継ぎ（HAND_OFF）

最終更新: **2026-11-08 12:10 JST**
更新担当: cqrs-es coordinator（Orca）
この文書は、担当者がコンテキストやレート制限で交代しても、次の担当がこの 1 枚で引き継げるようにするためのものです。実装の詳細は各 repo の issue / PR と `docs/acceptance-criteria.md` を参照してください。

## 0. 目的と対象

- 対象: `cqrs-es-example`（ハブ）と 6 つの言語別 repo（go / java / js / rs / scala / kotlin）。
- 目標: 全言語で ESA v5（go / java / kotlin / scala は 5.1.0、js / rs は 5.2.0）、DynamoDB 3 テーブル、envelope API、共通 GraphQL（GroupChat）、Write API / Read API / Read Model Updater（RMU）、docker-compose + LocalStack の e2e、main CI 緑。
- 既存データの移行は不要（サンプル）。

## 1. いまの状態（サマリ）

- **M0 受け入れ基準**: 完了（ハブ PR #6、設計判断 a〜k 確定）。
- **M1 赤 CI 修正**: 完了（go #673、java #589 をマージし、main CI 緑）。後続の go #672（staticcheck 整理）と #687（lint ジョブへの staticcheck v0.8.2 追加）も完了。
- **M2 ESA v5 移行**: go / java / js / rs の移行と復旧、scala の復旧、kotlin / scala の再同期順序修正が main にマージ済み。go は #675〜#679 + 復旧 #681、java は #618〜#622 + 復旧 #624、js は #831〜#835 + 復旧 #837 + Lambda 起動時 Scan #841、rs は #598・#599・#601・#602 + 復旧 #603、kotlin は #4〜#6 + 復旧 #8 + 順序修正 #11、scala は #5〜#7 + 復旧 #9 + 順序修正 #12。**M2 のコード完了条件は全言語で満たしている**。Renovate 整理 issue（java #588 / js #830 / rs #551）は停止方針により保留のため、各 M2 マイルストーンは open のまま。
- **M2 復旧（A 案）**: 2026-11-07 23:37 加藤さん決定。M2 を「head Streams 移行」と「復旧（再試行・DLQ・再同期）」に分離し、復旧を言語ごとの issue で実装した。追跡のハブ #13 は全言語完了でクローズ済み。go #680 / java #623 / js #836 / rs #600 / kotlin #7 / scala #8 はすべてクローズ済み。
- **M2 後続（本番適用・堅牢化）**: go #684 / java #627 / js #840 / rs #604 / go #672（staticcheck）/ kotlin #10 / scala #11 はすべて PR マージ済みでクローズ。go #29（ローカルのコンテナ検証まで）も PR #688 で完了しクローズ。**実装系の残作業はない**。実 AWS デプロイ、Secrets Manager、SNS 通知先、アラート通知先などの運用設定のみが残る（本番デプロイはオーナー承認が必要）。go の CI lint への staticcheck 追加は加藤さん決定（v0.8.2 固定）に沿って PR #687 で完了。
- **BIGINT 化（T-9）**: js PR #843（`6dadc460`）と scala PR #18（`820fa1d9`）で、`last_processed_sequence_number` を 4 テーブルとも BIGINT NOT NULL に拡張。migration 5、down の注意、T-9 上限検査、境界テストを実装済み。
- **Lambda 配布物サイズ計測**: java PR #631、kotlin PR #18、scala PR #19 で計測と CI ログを追加。java 119.16 MiB / 展開後 128.96 MiB、kotlin 121.30 MiB / 131.50 MiB、scala 118.67 MiB / **293.55 MiB**。scala は Lambda の展開後 250 MB 上限を超えるが、2026-11-08 10:47 の加藤さん裁定で **ESA JVM 系を 5 成果物に分割（5.2.0）** することになっており、scala の上限超過は ESA 5.2.0 の分割を待つ。レイヤー分割・依存除外はしない。
- **Renovate は停止中（2026-11-07 19:33 加藤さん決定）**: 使用量削減のため Renovate は off。既存 Renovate PR の修正・リベース・整理・クローズは禁止（WIP 超過解消目的でも）。作業者を Renovate PR に着手させない。マージは Grok Bot が CI 全緑＋未対応コメントなしのものだけ行う。WIP 超過が Renovate だけなら新規着手せず待機。**この停止中の方針は acceptance-criteria §10.h（patch 自動マージ、minor/major 手動）より優先する。**
- **M3 Kotlin**: **完了**。#1（骨格 + command、PR #4）、#2（Read API + RMU、PR #5）、#3（compose + e2e + CI、PR #6）、復旧 A 案（PR #8）、再同期順序（PR #11）、Lambda/SQS デプロイ資産（PR #12）、24 時間超停止の運用手順・アラート（PR #14）、Scan 中も購読を継続する修正（PR #16）、Lambda サイズ計測（PR #18）。issue #1〜#3・#7・#9・#10・#13・#15・#17 とマイルストーン、ハブ #4 はクローズ済み。e2e のホストポート固定 bind の修正（issue #19 / PR #20）は CI 実行中。
- **M4 Scala**: **完了**。scala #5 / #6 / #7 に加え、復旧 A 案（PR #9）、再同期順序（PR #12）、Lambda/SQS デプロイ資産（PR #13）、障害注入 e2e（PR #15）、BIGINT 化（PR #18）、Lambda サイズ計測（PR #19）。issue #8・#10・#11・#14・#16・#17 をクローズ済み。

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
  - #3 [M2] 追跡: ESA v5 移行（rs → js → java → go） https://github.com/j5ik2o/cqrs-es-example/issues/3 — Renovate 整理の保留を記録して残す
- クローズ済みの追跡: #4（M3 Kotlin、完了）、#13（M2 復旧、全言語完了）
- マイルストーン: M2 が open（Renovate 整理 issue の保留のため）。M0 / M1 / M3 / M4 は closed。

### go `j5ik2o/cqrs-es-example-go`

- open PR: #690（#689 の後続改善）— CI 実行中
- open issue:
  - #689 [M2 後続] ローカル検証の後続改善（定期実行の再処理テスト等） https://github.com/j5ik2o/cqrs-es-example-go/issues/689 — PR #690 で対応中
  - #7 Dependency Dashboard https://github.com/j5ik2o/cqrs-es-example-go/issues/7
- 補足: #666〜#671・#680・#682・#684・#672・#29（ローカル検証）・#32（対象外）は完了・クローズ済み。PR #675〜#679・#573・#681（復旧）・#683（本番適用資産）・#685（定期実行・件数上限・監視）・#686（staticcheck 整理）・#687（CI lint への staticcheck v0.8.2 追加）・#688（ローカルコンテナ検証）はマージ済み。M2 マイルストーンは closed。

### java `j5ik2o/cqrs-es-example-java`

- open PR（Renovate 3 本のみ。停止方針により触らない）:
  - #616 Spring Boot v4 https://github.com/j5ik2o/cqrs-es-example-java/pull/616
  - #608 java-jdk v25 https://github.com/j5ik2o/cqrs-es-example-java/pull/608
  - #559 Gradle 9（close 候補） https://github.com/j5ik2o/cqrs-es-example-java/pull/559
- マージ済み（M2）: #618（ESA v5.1.0 / 3 テーブル）、#619（Write API）、#620（Read API）、#621（RMU head Streams）、#622（e2e）、#624（復旧 A 案）、#626（Lambda/SQS デプロイ資産）、#629（Lambda 配布物・Secrets Manager・監視、`3b31c755`）、#631（Lambda サイズ計測、`2a0d1447`）。issue #583〜#587・#623・#625・#627・#630 はクローズ済み。
- open issue:
  - #588 [M2] Renovate PR を整理 https://github.com/j5ik2o/cqrs-es-example-java/issues/588 — **Renovate 停止方針で保留**

### js `j5ik2o/cqrs-es-example-js`

- open PR（Renovate 3 本のみ。停止方針により触らない）:
  - #819 functions-framework v5 https://github.com/j5ik2o/cqrs-es-example-js/pull/819
  - #812 @types/node https://github.com/j5ik2o/cqrs-es-example-js/pull/812
  - #811 yargs v18 https://github.com/j5ik2o/cqrs-es-example-js/pull/811
- マージ済み（M2）: #831（ESA v5.2.0）、#832（Write API）、#833（Read API）、#834（RMU head Streams）、#835（e2e）、#837（復旧 A 案）、#839（payload 復号失敗の DLQ と SQS 通知）、#841（Lambda 起動時 Scan・DLQ バッチ・SQS 消費側、`2288a2f8`）、#843（BIGINT 化・T-9、`6dadc460`）。issue #825〜#829・#836・#838・#840・#842 はクローズ済み。
- open issue:
  - #830 [M2] Renovate 整理 https://github.com/j5ik2o/cqrs-es-example-js/issues/830 — **Renovate 停止方針で保留**

### rs `j5ik2o/cqrs-es-example-rs`

- open PR（Renovate 5 本のみ。停止方針により触らない）:
  - #597 Terraform aws v6 https://github.com/j5ik2o/cqrs-es-example-rs/pull/597
  - #592 aws_lambda_events v1 https://github.com/j5ik2o/cqrs-es-example-rs/pull/592
  - #591 mysql v26 https://github.com/j5ik2o/cqrs-es-example-rs/pull/591
  - #590 Helm mysql v14 https://github.com/j5ik2o/cqrs-es-example-rs/pull/590
  - #582 sqlx 0.9 https://github.com/j5ik2o/cqrs-es-example-rs/pull/582
- マージ済み（M2）: #598（RMU head Streams）、#599（Write API）、#601（Read API、`f958c548`）、#602（e2e、`e5d22276`）、#603（復旧 A 案、`a5ee641e`）、#605（SQS ワーカー運用・監視、`0f1ea051`）。issue #547〜#550・#600・#604 はクローズ済み。
- open issue:
  - #551 [M2] Renovate 整理 https://github.com/j5ik2o/cqrs-es-example-rs/issues/551 — **Renovate 停止方針で保留**

### scala `j5ik2o/cqrs-es-example-scala`

- open PR: 0
- open issue: 0
- マージ済み: #5（Write API）、#6（Read API / RMU）、#7（head Streams / e2e）、#9（復旧 A 案、`03d1951a`）、#12（再同期順序、`cdcf400e`）、#13（Lambda/SQS デプロイ資産、`8640751a`）、#15（障害注入 e2e、`550de5d0`）、#18（BIGINT 化、`820fa1d9`）、#19（Lambda サイズ計測、`d54bb4a8`）。issue #2〜#4・#8・#10・#11・#14・#16・#17 はクローズ済み。
- 残作業: 実 AWS デプロイ。Lambda 配布物は展開後 293.55 MiB で 250 MB 上限を超えるが、ESA JVM 系の 5 成果物分割（5.2.0、2026-11-08 10:47 加藤さん裁定）を待つ。レイヤー分割・依存除外はしない。
- M4 マイルストーン: closed。

### kotlin `j5ik2o/cqrs-es-example-kotlin`

- open PR: #20（e2e のホストポート固定 bind 修正）— CI 実行中
- open issue:
  - #19 [M3 後続] e2e のホストポート固定バインドをやめる https://github.com/j5ik2o/cqrs-es-example-kotlin/issues/19 — PR #20 で対応中
- マージ済み: #4（骨格 + command + Write API、`f2564d72`）、#5（Read API + RMU、`461c6171`）、#6（compose + e2e + CI、`ea55da91`）、#8（復旧 A 案、`2ca6158b`）、#11（再同期順序、`5272c5c1`）、#12（Lambda/SQS デプロイ資産、`ac3ec0d7`）、#14（24 時間超停止の運用手順・アラート、`f1925bf8`）、#16（Scan 中も購読を継続、`9200537a`）、#18（Lambda サイズ計測、`fc279e74`）。issue #1〜#3・#7・#9・#10・#13・#15・#17 はクローズ済み。
- M3 マイルストーン: closed。
- 残作業: 実 AWS デプロイとアラーム通知先、MySQL 認証情報のローテーション手順。配布物は展開後 131.50 MiB で上限内のため、レイヤー分割は不要。

## 4. 作業中の Orca 端末・worktree・担当 issue

- **go #689（PR #690）**: worktree `m2-go-688-followup` / terminal `term_ec3f76b2-ad3e-47bd-ae81-a8a60e721e80` / task `task_7676049216d7`。
- **kotlin #19（PR #20）**: worktree `m3-kotlin-e2e-ports` / terminal `term_89ddb327-9688-4119-b249-d0cffb9b4fa6` / task `task_5b01a6d42656`。
- 上記 2 本以外に実装系の open PR（Renovate 除く）はない。
- go #687（PR #687）、kotlin #10（PR #12）、scala #11（PR #13）、go #672（PR #686）、kotlin #9（PR #11）、scala #10（PR #12）、rs #604（PR #605）、js #840（PR #841）、kotlin #7（PR #8）、java #627（PR #629）、go #684（PR #685）、scala #8（PR #9）、kotlin #13（PR #14）はすべてマージ済みで、各 worktree は削除済み。リモート枝は残す。
- Renovate PR は停止方針により触らない。
- 完了済み worker の端末は指示により close せず orphaned のまま: `term_18bfc47d`（go #667）、`term_dd421d56`（go #668）、`term_7d24356a`（go #669）、`term_313c97bc`（go #679）、`term_ea435055`（go #671）、`term_df43c75b`（go #573）、`term_f28d70c0`（scala #7）、`term_21227747`（scala #3 検証）、`term_22ca3215`（java #614）、`term_20839d9c`（js #823）。
- 端末上限対応で idle シェルを閉じた: scala main `term_6aee73dd`、hub `term_4f1d95f0`。

## 5. 判断待ち

- **rs #591（mysql v26）**: Renovate PR。変更が `tools/docker-compose/docker-compose-databases.yml` のみで、ci.yml の `paths-ignore: tools/**` により CI が走らない。healthcheck のローカル修正（`mysqladmin ping`）と commit `20707fc` は保持するが、**Renovate 停止中は push・paths-ignore 変更・admin merge をしない**。進めるには加藤さんによる明示的な方針変更が必要。マージは Renovate 停止方針（Grok Bot が CI 全緑＋未対応コメントなしのみ）に従う。#592 の `local.rs` も未コミットのまま保持する。
- **close 候補（勝手に close しない。加藤さんの判断待ち）**
  - java #559（Gradle 9）
  - js #819 / #812 / #811
- 設計判断 a〜k（M0）は **2026-11-06 に加藤さんが推奨どおり確定**。`docs/acceptance-criteria.md` §10 は「判断待ち」として列挙しているが、内容は確定済みとして読む。確定の記録はハブ issue #1 のコメントを参照: https://github.com/j5ik2o/cqrs-es-example/issues/1
- 復旧の設計論点（リトライ上限、DLQ、24 時間超停止後の再同期）は **2026-11-07 23:37 に A 案で確定**。go / java / js / rs / kotlin / scala のすべてで実装・マージ済み。**kotlin / scala のローカル reader の再同期順序（購読再開→並行 Scan）も PR #11 / #12 で修正済み**。
- **kotlin の Scan 中の購読**: 2026-11-07 23:37 確定の A 案（購読を再開したまま並行 Scan → journal から欠番補完）で決着済み。PR #16 で「Scan 中も購読を継続する」形に修正済み。長時間の Scan 中も読み取りを続け、失敗時はバックオフして再開する。
- **js #840 の Lambda 起動時 Scan** は PR #841 で実装済み。M2 の完了条件を満たす。
- **scala の DY-15 再同期** は PR #9、順序修正は PR #12 で実装済み。Lambda/SQS デプロイ資産（#11）と障害注入 e2e（#14）も完了。
- **scala の Lambda 配布物サイズ**: 展開後 293.55 MiB で Lambda の 250 MB 上限を超える。2026-11-08 10:47 の加藤さん裁定で **ESA JVM 系を 5 成果物に分割（5.2.0）** することになっており、scala は ESA 5.2.0 の分割を待つ。レイヤー分割・依存除外はしない。
- **go の CI の staticcheck**: 2026-11-08 07:56 に加藤さんが決定済み（v0.8.2 固定、既存 lint ジョブに追加、別ジョブにしない）。PR #687 で実装済みで、go の main の lint ジョブに staticcheck のインストールと実行があることを確認済み。

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

1. **go #690 / kotlin #20**: CI と未解決レビューコメントを確認し、Grok Bot のマージ判断を待つ。
2. **M2 / M3 / M4 の実装**: 全言語で完了。Lambda/SQS デプロイ資産・BIGINT 化・サイズ計測も完了。
3. **判断待ち**: 実 AWS デプロイ、Secrets Manager、SNS / アラーム通知先。scala の 293.55 MiB 超過は ESA JVM 系 5 成果物分割（5.2.0）待ちで、レイヤー分割・依存除外はしない。
4. **Renovate 整理 issue**（java #588 / js #830 / rs #551）は停止方針で保留。
5. **現在は停止中**: §9 の再開手順に従う。

## 8. 参照

- 計画: `/Users/j5ik2o/orca/handoff/cqrs-es/cqrs-es-plan.md`
- 受け入れ基準: https://github.com/j5ik2o/cqrs-es-example/blob/main/docs/acceptance-criteria.md
- ESA v5 DynamoDB 仕様: https://github.com/j5ik2o/event-store-adapter/blob/main/docs/spec/storage/dynamodb.ja.md
- ADR-0002: https://github.com/j5ik2o/event-store-adapter/blob/main/docs/adr/0002-separate-aggregate-head-from-snapshot.md

## 9. 停止中（2026-11-08 12:01 加藤さん指示）

**理由**: ひと段落したら HAND_OFF を書いて止まる。新しい issue には着手しない。開いている PR は CI と未対応コメントが片づくまで仕上げ、マージは Grok Bot が行う。作業者は起動せず、新しい指示があるまで待機する。

### repo ごとの状態

| repo | 実装 PR | 主な残り | main CI |
|---|---|---|---|
| go | #690（#689 後続改善、CI 実行中） | #689（PR #690 で対応中）、#7 | 緑 |
| java | なし（Renovate 3 本のみ） | #588（Renovate 整理、保留） | 緑 |
| js | なし（Renovate 3 本のみ） | #830（Renovate 整理、保留） | 緑 |
| rs | なし（Renovate 5 本のみ） | #551（Renovate 整理、保留） | 緑 |
| scala | なし | なし。Lambda 展開後 293.55 MiB は ESA JVM 系 5 成果物分割（5.2.0）待ち | 緑 |
| kotlin | #20（#19 e2e ポート修正、CI 実行中） | #19（PR #20 で対応中） | 緑（再実行で成功） |
| hub | なし | #3（M2 追跡、Renovate 保留の記録） | 対象外 |

### 残りの issue

- go #689: ローカル検証の後続改善（PR #690）
- kotlin #19: e2e のホストポート固定 bind 修正（PR #20）
- go #7 / java #588 / js #830 / rs #551: Renovate 関連（停止方針で保留）
- hub #3: M2 追跡（Renovate 整理 issue が保留のため残す）

### 再開手順

1. `go #690` と `kotlin #20` の CI・未解決レビューコメントを確認する。全緑・未解決 0 なら Grok Bot がマージする。
2. マージ後に各 issue（#689 / #19）をクローズし、worktree `m2-go-688-followup` / `m3-kotlin-e2e-ports` を、未コミット・未プッシュなしを確認してローカル破棄する（リモート枝は残す）。
3. 判断待ち（実 AWS デプロイ・Secrets Manager・SNS 通知先、Renovate 整理）は加藤さんの指示があるまで着手しない。scala の 293.55 MiB 超過は ESA 5.2.0 の 5 成果物分割待ちで、レイヤー分割・依存除外はしない。
4. 作業者は起動しない。Renovate PR は触らない。

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
- 2026-11-08 04:00 JST: M2 復旧の全言語マージ（go #681 / java #624 / js #837 / rs #603）と e2e（rs #602）、本番適用資産（go #683 / java #626 / js #839）を反映。M2 後続を go #684 / java #627 / js #840 / rs #604 に整理。kotlin は #4 / #5 マージ済みで #3 を着手中。go / java の worktree は端末が残るため破棄保留。
- 2026-11-08 06:55 JST: 全言語の復旧完了を反映（kotlin #8、scala #9、js #841、rs #605、go #685、java #629）。hub #13 / #4 と kotlin M3 マイルストーンをクローズ。M2 後続はすべて完了し、残る実装 issue は go #672（staticcheck）のみ。scala / kotlin の Lambda/SQS デプロイ資産は残作業として区別して記載。kotlin / scala のローカル reader は Scan 完了後に購読を開始するため、24 時間超の Scan では反映漏れの可能性があることを明記。マージ済み worktree を削除済み。
- 2026-11-08 08:05 JST: kotlin #11 / scala #12 の再同期順序修正、go #686（staticcheck）のマージを反映。ローカル reader の順序課題は解消済みに更新。kotlin #10 / scala #11 の Lambda/SQS デプロイ資産を着手中として記録。go の CI への staticcheck 追加は判断待ちとして明記。
- 2026-11-08 08:20 JST: go の open issue（#32 / #29 / #7）を明記。kotlin の購読開始直後の Scan 中に Streams 読み取りが長時間止まった場合の残課題を §5 に追記。
- 2026-11-08 08:45 JST: go #687（CI staticcheck）、scala #13（デプロイ資産）、hub #17、kotlin #12（デプロイ資産）のマージを反映。kotlin #10 / scala #11 はクローズ済み。実装系の残作業はなく、残るのは本番デプロイ・Secrets・通知先などの運用設定。全実装 worktree を削除済み。
- 2026-11-08 09:10 JST: kotlin の Scan 中の購読停止は判断待ちではなく、A 案（購読再開→並行 Scan→journal 補完）で決着済みとして記述を修正。issue #15 で Scan 中も購読を継続する修正を行うことを明記。
- 2026-11-08 09:25 JST: kotlin #14（24h 超停止の運用手順・アラート、`f1925bf8`）のマージを反映。kotlin #15 と scala #14（障害注入 e2e / PR #15）を進行中として記録。
- 2026-11-08 09:40 JST: scala #15（障害注入 e2e、`550de5d0`）と kotlin #16（Scan 中も購読を継続、`9200537a`）のマージを反映。issue #14 / #15 クローズ、全実装 worktree を削除・端末もクローズ。着手可能な非デプロイ issue は残っていない。
- 2026-11-08 12:05 JST: 加藤さん指示（12:01）により停止。js #843 / scala #18 の BIGINT 化、java #631 / kotlin #18 / scala #19 の Lambda サイズ計測、go #688 のローカル検証、kotlin #14 / #16 を反映。go #29 / #32 はクローズ。go #690（#689）と kotlin #20（#19）を CI 実行中として記録し、§9 に停止理由・repo 状態・残 issue・再開手順を追加。
- 2026-11-08 12:10 JST: 記述訂正。go の staticcheck は 07:56 に決定済み・PR #687 で実装済み（v0.8.2、既存 lint ジョブ）。scala の 293.55 MiB 超過は未決定ではなく、10:47 の加藤さん裁定で ESA JVM 系 5 成果物分割（5.2.0）を待つ扱い。レイヤー分割・依存除外はしない。
