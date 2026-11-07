# cqrs-es-example 群 引継ぎ（HAND_OFF）

最終更新: **2026-11-07 16:10 JST**
更新担当: cqrs-es coordinator（Orca）
この文書は、担当者がコンテキストやレート制限で交代しても、次の担当がこの 1 枚で引き継げるようにするためのものです。実装の詳細は各 repo の issue / PR と `docs/acceptance-criteria.md` を参照してください。

## 0. 目的と対象

- 対象: `cqrs-es-example`（ハブ）と 6 つの言語別 repo（go / java / js / rs / scala / kotlin）。
- 目標: 全言語で ESA v5（go / java / kotlin / scala は 5.1.0、js / rs は 5.2.0）、DynamoDB 3 テーブル、envelope API、共通 GraphQL（GroupChat）、Write API / Read API / Read Model Updater（RMU）、docker-compose + LocalStack の e2e、main CI 緑。
- 既存データの移行は不要（サンプル）。

## 1. いまの状態（サマリ）

- **M0 受け入れ基準**: 完了（ハブ PR #6、設計判断 a〜k 確定）。
- **M1 赤 CI 修正**: 完了（go #673、java #589 をマージし、main CI 緑）。
- **M2 ESA v5 移行**: 進行中（rs / js / java / go）。**依存更新 PR のマージだけでは完了しない**（下記の head Streams 条件を参照）。
- **M3 Kotlin**: 未着手。**java の v5 版が骨格**のため、java M2 完了待ち。
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
- 対象 issue: **rs #549 / go #669 / java #586 / js #828**。scala は #3 として実装済み（PR #7）。
- 注意: js など MODIFY を無視している実装は、購読先を head に替えると 2 件目以降のイベントを取りこぼす。

### M3 Kotlin

- java の M2（v5 版の command / query / RMU）が完了してから着手する。
- kotlin #1（骨格 + command）→ #2（query/Read API + RMU）→ #3（compose + e2e + CI）の順。

### M4 Scala（完了）

- scala #2（ドメイン + Write API）→ #3（Read API + RMU、head Streams 条件を含む）→ #4（compose + LocalStack e2e + CI）を PR #5 / #6 / #7 でマージ済み。

## 3. repo ごとの open PR と issue

### ハブ `j5ik2o/cqrs-es-example`

- open PR: この HAND_OFF PR のみ。
- open issue:
  - #3 [M2] 追跡: ESA v5 移行（rs → js → java → go） https://github.com/j5ik2o/cqrs-es-example/issues/3
  - #4 [M3] 追跡: Kotlin 版の新規実装 https://github.com/j5ik2o/cqrs-es-example/issues/4
- マイルストーン: M2 / M3 が open。M0 / M1 / M4 は closed。

### go `j5ik2o/cqrs-es-example-go`

- open PR:
  - #573 Renovate mysql Docker tag v26（e2e FAILURE。原因は mysql 26.10 の healthcheck `mysql -e "quit"` が ERROR 1064。原因・最小修正案をコメント済み。**マージ・クローズしない**） https://github.com/j5ik2o/cqrs-es-example-go/pull/573
- open issue:
  - #668 [M2] Read API を v5 に適合させる（**作業中**） https://github.com/j5ik2o/cqrs-es-example-go/issues/668
  - #669 [M2] Read Model Updater を v5 に適合させる（**head Streams 条件**） https://github.com/j5ik2o/cqrs-es-example-go/issues/669
  - #670 [M2] docker-compose + LocalStack の e2e を整える https://github.com/j5ik2o/cqrs-es-example-go/issues/670
  - #671 [M2] Renovate PR を整理する https://github.com/j5ik2o/cqrs-es-example-go/issues/671
  - #672 [M1 後続] staticcheck の既存指摘 41 件 https://github.com/j5ik2o/cqrs-es-example-go/issues/672
- 補足: #666（ESA v5.1.0 更新）と #667（Write API）は完了・クローズ済み。

### java `j5ik2o/cqrs-es-example-java`

- open PR（WIP 5、超過）:
  - #616 Spring Boot v4（lint FAILURE） https://github.com/j5ik2o/cqrs-es-example-java/pull/616
  - #614 mysql v26（e2e FAILURE） https://github.com/j5ik2o/cqrs-es-example-java/pull/614
  - #608 java-jdk v25（lint FAILURE） https://github.com/j5ik2o/cqrs-es-example-java/pull/608
  - #559 Gradle 9（close 候補） https://github.com/j5ik2o/cqrs-es-example-java/pull/559
  - #400 localstack 4（close 候補） https://github.com/j5ik2o/cqrs-es-example-java/pull/400
- open issue:
  - #583 [M2] ESA v5.1.0 へ更新 https://github.com/j5ik2o/cqrs-es-example-java/issues/583
  - #584 [M2] Write API を v5 envelope に適合 https://github.com/j5ik2o/cqrs-es-example-java/issues/584
  - #585 [M2] Read API を v5 に適合 https://github.com/j5ik2o/cqrs-es-example-java/issues/585
  - #586 [M2] RMU を v5 に適合（**head Streams 条件**） https://github.com/j5ik2o/cqrs-es-example-java/issues/586
  - #587 [M2] docker-compose + LocalStack e2e https://github.com/j5ik2o/cqrs-es-example-java/issues/587
  - #588 [M2] Renovate PR を整理 https://github.com/j5ik2o/cqrs-es-example-java/issues/588

### js `j5ik2o/cqrs-es-example-js`

- open PR（WIP 6、超過。多くが FAILURE / 古い）:
  - #824 @types/aws-lambda https://github.com/j5ik2o/cqrs-es-example-js/pull/824
  - #823 [codex] Renovate PR のフォーマット補正 https://github.com/j5ik2o/cqrs-es-example-js/pull/823
  - #819 functions-framework v5 https://github.com/j5ik2o/cqrs-es-example-js/pull/819
  - #815 ts-jest https://github.com/j5ik2o/cqrs-es-example-js/pull/815
  - #812 @types/node https://github.com/j5ik2o/cqrs-es-example-js/pull/812
  - #811 yargs v18 https://github.com/j5ik2o/cqrs-es-example-js/pull/811
- open issue:
  - #825 [M2] ESA v5.2.0 へ更新 https://github.com/j5ik2o/cqrs-es-example-js/issues/825
  - #826 [M2] Write API https://github.com/j5ik2o/cqrs-es-example-js/issues/826
  - #827 [M2] Read API https://github.com/j5ik2o/cqrs-es-example-js/issues/827
  - #828 [M2] RMU（**head Streams 条件**） https://github.com/j5ik2o/cqrs-es-example-js/issues/828
  - #829 [M2] compose + e2e https://github.com/j5ik2o/cqrs-es-example-js/issues/829
  - #830 [M2] Renovate 整理 https://github.com/j5ik2o/cqrs-es-example-js/issues/830

### rs `j5ik2o/cqrs-es-example-rs`

- open PR（WIP 5、超過）:
  - #593 downcast-rs v2 https://github.com/j5ik2o/cqrs-es-example-rs/pull/593
  - #592 aws_lambda_events v1 https://github.com/j5ik2o/cqrs-es-example-rs/pull/592
  - #591 mysql v26 https://github.com/j5ik2o/cqrs-es-example-rs/pull/591
  - #590 Helm mysql v14 https://github.com/j5ik2o/cqrs-es-example-rs/pull/590
  - #582 sqlx 0.9（FAILURE） https://github.com/j5ik2o/cqrs-es-example-rs/pull/582
- open issue:
  - #547 [M2] Write API https://github.com/j5ik2o/cqrs-es-example-rs/issues/547
  - #548 [M2] Read API https://github.com/j5ik2o/cqrs-es-example-rs/issues/548
  - #549 [M2] RMU（**head Streams 条件**） https://github.com/j5ik2o/cqrs-es-example-rs/issues/549
  - #550 [M2] compose + e2e https://github.com/j5ik2o/cqrs-es-example-rs/issues/550
  - #551 [M2] Renovate 整理 https://github.com/j5ik2o/cqrs-es-example-rs/issues/551
- 補足: #546（ESA v5.2.0 更新）は PR #557 で完了・クローズ済み。#557 で RMU の初期 head Streams 対応も入っている。

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

- **go #668（Read API v5）**
  - worktree: `m2-go-read-api`（`/Users/j5ik2o/orca/workspaces/cqrs-es-example-go/m2-go-read-api`）
  - worker terminal: `term_dd421d56-0a46-453f-a951-d060272b691b`
  - task: `task_d452ad13ba89`
  - 起動コマンド: `ollama launch claude --model deepseek-v4.1-flash:cloud -- --dangerously-skip-permissions`
- ほかに稼働中の worker はいない。
- 端末上限対応で idle シェルを閉じた: scala main `term_6aee73dd`、hub `term_4f1d95f0`。
- 完了済み worker の端末は指示により close せず orphaned のまま: `term_18bfc47d`（go #667）、`term_f28d70c0`（scala #7）。

## 5. 判断待ち

- **設計判断（実装は止めない。PR で論点として報告）**
  - RMU のリトライ上限
  - DLQ の要否・方式
  - 24 時間を超えて RMU が止まったときの再同期手順
- **close 候補（勝手に close しない。加藤さんの判断待ち）**
  - java #559（Gradle 9）/ #400（localstack 4）
  - js #824 / #823 / #819 / #815 / #812 / #811
  - go #573（mysql v26。切り分けコメント済み）
- 設計判断 a〜k（M0）は確定済み。`docs/acceptance-criteria.md` を参照。

## 6. 加藤さんの決定・禁止事項

- **WIP は repo ごと**: 開いている PR は 1 repo 3 本まで（ドラフト含む）。作業者は 1 repo 1 本。repo が違えば並行可。
- **Bugbot 確認必須**: Bugbot / CodeRabbit の未解決スレ・未対応コメントがある PR はマージしない。Codex の usage limit と CodeRabbit の rate-limit のコメントは指摘扱いしない。
- **M2 完了条件**: 上記 2 章の head Streams 条件を満たすこと。依存更新 PR のマージだけでは完了にしない。
- **禁止**:
  - AGENTS.md の編集
  - docker prune
  - 他担当のコンテナ・画像・worktree・terminal の削除
  - 公開・リリース
  - force push / rebase（同期は `git merge origin/main` または GitHub の Update branch を使う）
  - **PR #83 の再実装**
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

1. **go #668（進行中）**: Read API v5 適合。完了後に #669（RMU head Streams、M2 完了条件）→ #670（e2e）→ #671（Renovate 整理）。
2. **rs**: open PR を減らして WIP 3 以下にする。その後 #547 Write → #548 Read → #549 RMU（head Streams 条件）→ #550 e2e → #551 Renovate。#582（FAILURE）は触らない。
3. **js**: open PR を減らして WIP 3 以下にする。その後 #825 ESA 更新 → #826 Write → #827 Read → #828 RMU（head Streams）→ #829 e2e → #830 Renovate。
4. **java**: open PR を減らして WIP 3 以下にする。その後 #583 ESA 更新 → #584 Write → #585 Read → #586 RMU（head Streams）→ #587 e2e → #588 Renovate。
5. **kotlin M3**: java M2（特に v5 版の骨格）完了後に #1 → #2 → #3。
6. **HAND_OFF**: 最長 1 時間おき、およびマイルストーン完了・方針変更時に更新。
7. **設計論点**: リトライ上限、DLQ、24 時間超停止後の再同期を決める。

## 8. 参照

- 計画: `/Users/j5ik2o/orca/handoff/cqrs-es/cqrs-es-plan.md`
- 受け入れ基準: https://github.com/j5ik2o/cqrs-es-example/blob/main/docs/acceptance-criteria.md
- ESA v5 DynamoDB 仕様: https://github.com/j5ik2o/event-store-adapter/blob/main/docs/spec/storage/dynamodb.ja.md
- ADR-0002: https://github.com/j5ik2o/event-store-adapter/blob/main/docs/adr/0002-separate-aggregate-head-from-snapshot.md

## 更新履歴

- 2026-11-07 16:10 JST: 初版。M0/M1/M4 完了、M2 進行中、M3 待ちの状態を記録。
