# cqrs-es-example 群の引継ぎ

確認日時: 2026-11-09 15:45 JST
ローカル e2e 実行: 2026-11-09 19:50〜20:05 JST
必須チェックの設定: 2026-11-09 21:00〜23:00 JST
対象: ハブと go / java / js / rs / scala / kotlin の6言語実装。

## ゴールと対象範囲

2026-11-09 のユーザー指示により、ゴールは全6言語がローカルの Docker Compose と LocalStack で稼働すること。Write API から書き込み、RMU による MySQL のリードモデル更新を経て、Read API で結果を取得できることを確認する。

実 AWS へのデプロイ、Secrets Manager・SNS・アラーム通知先の設定、Lambda 配布物のサイズ適合、パッケージ公開・リリースは完成条件に含めない。Scala のローカル稼働を ESA の成果物分割待ちにはしない。

## 根拠と確認範囲

実装の SSoT は各リポジトリの git 履歴とコード。この文書は確認時点の索引であり、完了判定の根拠を複製しない。コードへのリンクは確認したコミットに固定する。PR・issue の開閉と CI の成否は GitHub の実データを使う。

今回は各 repo の `origin/main` を fetch し、依存定義、RMU の主要処理、関連テスト、Compose の e2e 入口と通常 CI を照合した。そのうえで、同じ固定コミットを worktree に展開し、Compose e2e を6言語とも実行した（結果は「[ローカル稼働の確認結果](#ローカル稼働の確認結果)」）。受け入れ基準の §7 以外の全項目の再監査は行っていない。CI が成功したこととローカルで再現できたことは別の根拠として扱う。

受け入れ要件は [acceptance-criteria.md](acceptance-criteria.md) を参照する。同文書 §10 の「判断待ち」は初期の設計検討記録を含むため、現在の進捗判定には使わない。過去の経過・判断の記録は [旧 HAND_OFF](https://github.com/j5ik2o/cqrs-es-example/blob/a6abee66f69e043dac083ed6bbbde1492921e3c1/docs/HAND_OFF.md) と git 履歴に残る。

## 確認した main と通常 CI

ESA のバージョンは各行の依存定義から確認した。CI はそのコミットを対象とする実行で、すべて success。Java の `Bump-Version` は公開用の別ワークフローであり、ローカル稼働の完成条件には含めない。

| repo | 確認コミット | ESA の依存定義 | 通常 CI |
|---|---|---|---|
| go | `84968d9009f2` | [go.mod: v5.1.0](https://github.com/j5ik2o/cqrs-es-example-go/blob/84968d9009f2afaef94b0be363bf390f58c45ff5/go.mod) | [success](https://github.com/j5ik2o/cqrs-es-example-go/actions/runs/42442316505) |
| java | `8727cb6c9973` | [build.gradle.kts: 5.1.0](https://github.com/j5ik2o/cqrs-es-example-java/blob/8727cb6c99739ac08863a5602b24c0c506f24567/build.gradle.kts) | [success、JDK 17 と 25](https://github.com/j5ik2o/cqrs-es-example-java/actions/runs/42539789453) |
| js | `2ecafa5c95c5` | [packages/rmu/package.json: 5.2.0](https://github.com/j5ik2o/cqrs-es-example-js/blob/2ecafa5c95c5fcad8edb48fb8748cf97c77c31f4/packages/rmu/package.json) | [success](https://github.com/j5ik2o/cqrs-es-example-js/actions/runs/42491078342) |
| rs | `538f2e8d31dc` | [Cargo.toml: 5.2.0](https://github.com/j5ik2o/cqrs-es-example-rs/blob/538f2e8d31dc242144ad9eacea802b75a4b18f93/Cargo.toml) | [success](https://github.com/j5ik2o/cqrs-es-example-rs/actions/runs/42554521844) |
| scala | `d54bb4a8400b` | [build.sbt: 5.1.0](https://github.com/j5ik2o/cqrs-es-example-scala/blob/d54bb4a8400be3886d3e9cb173df1b24d54bcccf/build.sbt) | [success](https://github.com/j5ik2o/cqrs-es-example-scala/actions/runs/42437625464) |
| kotlin | `9668f0257928` | [build.gradle.kts: 5.1.0](https://github.com/j5ik2o/cqrs-es-example-kotlin/blob/9668f02579280e70e22996d5cf7d449f5dcb9628/build.gradle.kts) | [success](https://github.com/j5ik2o/cqrs-es-example-kotlin/actions/runs/42442320708) |

ハブの確認基点は `a6abee66f69e`。ハブ自体はドキュメントのみを持つ。

## コードから辿る実装

各言語には Write API、Read API、RMU と復旧関連の実装・テストが main に存在する。M0〜M4 の経過やマージ済み PR の全件一覧は履歴から辿り、ここでは再開時に読む場所を示す。

| repo | RMU・復旧の入口 | テスト・Compose e2e |
|---|---|---|
| go | [pkg/rmu](https://github.com/j5ik2o/cqrs-es-example-go/tree/84968d9009f2afaef94b0be363bf390f58c45ff5/pkg/rmu): `applier.go`、`recovery.go`、`resync.go` | [テスト・Compose e2e の CI 定義](https://github.com/j5ik2o/cqrs-es-example-go/blob/84968d9009f2afaef94b0be363bf390f58c45ff5/.github/workflows/ci.yml)、[e2e 検証スクリプト](https://github.com/j5ik2o/cqrs-es-example-go/blob/84968d9009f2afaef94b0be363bf390f58c45ff5/tools/e2e-test/verify-group-chat.sh) |
| java | [rmu](https://github.com/j5ik2o/cqrs-es-example-java/tree/8727cb6c99739ac08863a5602b24c0c506f24567/rmu): `Applier`、`RecoveryService`、`Resyncer` | [テスト・Compose e2e の CI 定義](https://github.com/j5ik2o/cqrs-es-example-java/blob/8727cb6c99739ac08863a5602b24c0c506f24567/.github/workflows/ci.yml)、[e2e 検証スクリプト](https://github.com/j5ik2o/cqrs-es-example-java/blob/8727cb6c99739ac08863a5602b24c0c506f24567/tools/e2e-test/verify-group-chat.sh) |
| js | [packages/rmu/src](https://github.com/j5ik2o/cqrs-es-example-js/tree/2ecafa5c95c5fcad8edb48fb8748cf97c77c31f4/packages/rmu/src): `update-read-model.ts`、`recovery.ts`、`stream-recovery.ts` | [テスト・Compose e2e の CI 定義](https://github.com/j5ik2o/cqrs-es-example-js/blob/2ecafa5c95c5fcad8edb48fb8748cf97c77c31f4/.github/workflows/ci.yml)、[e2e 検証スクリプト](https://github.com/j5ik2o/cqrs-es-example-js/blob/2ecafa5c95c5fcad8edb48fb8748cf97c77c31f4/tools/e2e-test/verify-group-chat.sh) |
| rs | [modules/rmu/src](https://github.com/j5ik2o/cqrs-es-example-rs/tree/538f2e8d31dc242144ad9eacea802b75a4b18f93/modules/rmu/src): `applier.rs`、`recovery.rs`、`resync.rs` | [テスト・Compose e2e の CI 定義](https://github.com/j5ik2o/cqrs-es-example-rs/blob/538f2e8d31dc242144ad9eacea802b75a4b18f93/.github/workflows/ci.yml)、[e2e 検証スクリプト](https://github.com/j5ik2o/cqrs-es-example-rs/blob/538f2e8d31dc242144ad9eacea802b75a4b18f93/tools/e2e-test/verify-group-chat.sh) |
| scala | [modules/rmu](https://github.com/j5ik2o/cqrs-es-example-scala/tree/d54bb4a8400be3886d3e9cb173df1b24d54bcccf/modules/rmu): `ReadModelEventApplier`、`RecoveryService`、`LocalRmuRunner` | [テスト・Compose e2e の CI 定義](https://github.com/j5ik2o/cqrs-es-example-scala/blob/d54bb4a8400be3886d3e9cb173df1b24d54bcccf/.github/workflows/ci.yml)、[e2e 検証スクリプト](https://github.com/j5ik2o/cqrs-es-example-scala/blob/d54bb4a8400be3886d3e9cb173df1b24d54bcccf/tools/e2e-test/verify-group-chat.sh) |
| kotlin | [rmu](https://github.com/j5ik2o/cqrs-es-example-kotlin/tree/9668f02579280e70e22996d5cf7d449f5dcb9628/rmu): `LocalRmuRunner`、`RecoveryService`、`Resyncer` | [テスト・Compose e2e の CI 定義](https://github.com/j5ik2o/cqrs-es-example-kotlin/blob/9668f02579280e70e22996d5cf7d449f5dcb9628/.github/workflows/ci.yml)、[e2e 検証スクリプト](https://github.com/j5ik2o/cqrs-es-example-kotlin/blob/9668f02579280e70e22996d5cf7d449f5dcb9628/tools/e2e-test/verify-group-chat.sh) |

履歴とコードで、旧版に残っていた次の「作業中」「保留」は解消済みと確認した。

- go #690 は `84968d9`、kotlin #20 は `9668f02` に取り込み済み。後者の e2e compose は API のホストポートを `ports: !reset []` で解除している。
- kotlin の `LocalRmuRunner` は購読開始・初回 poll の後に Scan し、購読は Scan 中も継続する。scala の `LocalRmuRunner` は poll と再同期を別スレッドで実行する。
- JS の `5_expand_sequence_number_to_bigint.up.sql` と Scala の `5_expand_last_processed_sequence_number.up.sql` は、位置行と3つのリードモデルテーブルの seq カラムを `BIGINT NOT NULL` に変更する。
- Java #559・#608・#616 は main に取り込み済み。現行コードは Gradle 9.9.0、Spring Boot 4.1.2、CI のテスト対象は JDK 17 と 25。JS #819 は `2ecafa5` に取り込み済み。
- Rust #592 は `7f050ca`、後続の Buildx 修正 #606 は `538f2e8` に取り込み済み。Go の lint ジョブには staticcheck v0.8.2 がある。

## ローカル稼働の確認結果

上表の確認コミットを worktree に展開し、各 repo の CI e2e ジョブと同じコマンドをローカルの Docker Compose と LocalStack で実行した。6言語すべてが終了コード 0 で完了した。実行環境は macOS arm64、Docker 29.9.2、Docker Compose v5.6.1、14 CPU、コンテナ用メモリ 8 GiB。

| repo | 実行したコマンド | 終了コード | 最後に成功した検証 |
|---|---|---|---|
| go | `make docker-compose-ci` | 0 | `Deploy asset verification passed` |
| java | `tools/e2e-test` の `make build`、`tools/scripts/docker-build.sh`、`tools/scripts/docker-compose-e2e-test.sh` | 0 | `End-to-end verification and restart replay succeeded` |
| js | `pnpm docker-build-e2e-test && pnpm docker-build && pnpm docker-compose-e2e-test` | 0 | `=== e2e completed ===` |
| rs | `tools/e2e-test` の `make build`、`docker-build-write-api-server.sh`、`docker-build-read-model-updater-local.sh`、`docker-build-read-api-server.sh`、`docker-compose-e2e-test.sh` | 0 | `e2e-test and post-restart replay succeeded` |
| scala | `tools/scripts/docker-compose-e2e-test.sh` | 0 | `PASS: RMU fault-injection e2e (5 scenarios)` |
| kotlin | `tools/scripts/docker-compose-e2e-test.sh` | 0 | `End-to-end verification and restart replay succeeded` |

[acceptance-criteria.md §7.3](acceptance-criteria.md#73-成功の判定) の4条件は、各スクリプトの成否と同じログで確認した。Write API の Mutation は `errors` を返さずに通り、Read API の Query は書き込み後の値を返し、リードモデルの反映待ちはタイムアウトせず、サービスは検証中の再起動シナリオ以外で停止していない。ログに出る `Restarting` は、いずれも再起動後の replay を確かめる各スクリプト自身の操作である。

この確認は「起動できた」ことの確認ではない。各スクリプトは作成・更新・削除の Mutation を投げ、リードモデルへの反映をポーリングし、Read API の Query が書き込み後の値を返すことを確かめてから終了する。go はデプロイ資産の検証、js は RMU を止めている間の書き込みと復帰、java・kotlin・rs は再起動後の replay、scala は障害注入の5シナリオまで含む。

実行ログは手元の `/tmp/e2e/<lang>.log` に残した。再現するときは各 repo の固定コミットを checkout し、上のコマンドを実行する。全言語に `tools/scripts/docker-compose-e2e-test.sh` があるが、事前ビルドの手順は言語ごとに異なるため、スクリプト単体の実行を共通手順とはしない。

java と rs のビルドスクリプトは `--builder amd-arm` を指定する。builder がない環境では、先に CI と同じ builder を作る。

```sh
docker buildx create --name amd-arm --driver docker-container --platform linux/amd64
docker buildx inspect amd-arm
```

各言語の e2e スクリプトは、固定のプロジェクト名（`cqrs-es-example-<lang>`）で `docker compose down -v --remove-orphans` を実行する。同じプロジェクト名の stack と MySQL のボリュームがあると、最初にそれらを削除する。worktree は Docker の資源を分離しないため、手元の開発用 stack と同じ Docker 環境で並行して動かさない。専用の Docker 環境で実行する。

### 必須チェックの設定（2026-11-09 追加）

[acceptance-criteria.md §7.4](acceptance-criteria.md#74-ci-で必須にするもの) に合わせて、6 repo の main で lint・test・e2e を必須チェックにした。go・java・rs の CI は `paths-ignore` を持っており、そのパスだけを触る Pull Request はチェックを1つも報告しない。必須チェックを先に入れるとそれらの PR が永久にマージできなくなるため、`paths-ignore` を外す PR を先に出してマージした。

| repo | 変更 | マージコミット | 必須にした check |
|---|---|---|---|
| go | [#691](https://github.com/j5ik2o/cqrs-es-example-go/pull/691) | `dc2d979` | `lint`、`test`、`e2e` |
| java | [#632](https://github.com/j5ik2o/cqrs-es-example-java/pull/632) | `5af6503` | `Run lint`、`Run tests with JDK 17`、`Run tests with JDK 25`、`e2e` |
| js | 設定のみ | `2ecafa5` | `Run lint`、`test (20.x)`、`e2e (20.x)` |
| rs | [#607](https://github.com/j5ik2o/cqrs-es-example-rs/pull/607) | `20ca43c` | `lint`、`test-infrastructure`、`test-command-domain`、`test-command-interface-adaptor-impl`、`test-command-processor`、`test-query-interface-adaptor`、`test-write-api-server`、`test-rmu`、`test-read-model-updater`、`test-read-api-server`、`e2e` |
| scala | strict の有効化のみ | `d54bb4a` | `lint`、`test`、`e2e` |
| kotlin | 設定のみ | `9668f02` | `Lint`、`Test`、`E2E` |

java と kotlin は main が未保護だったので protection を新しく作った。6 repo とも strict を有効にしたため、PR は main に追従してからでないとマージできない。確認は `gh api repos/j5ik2o/<repo>/branches/main/protection` の `required_status_checks.contexts` で行い、go `dc2d979`、java `5af6503`、rs `20ca43c` は main への push でも3ジョブが緑になることを確かめた。

この表のコミットは CI と保護設定だけの変更である。js・scala・kotlin はローカル e2e を実行した確認コミットと同じで、go・java・rs だけが異なる（ローカル e2e の確認コミットは `84968d9`、`8727cb6`、`538f2e8`）。コードへのリンクは引き続き上の表の確認コミットに固定している。

### 残る確認と差分

- §7.4 の必須ジョブは満たした。設定内容と根拠は前節に記す。
- LocalStack のイメージは java が 4.14.0、他5言語が 2.3.2 のまま。4.14.0 に揃える試みは go で行い、取りやめた。go の Compose でイメージだけ上げると、head テーブルのストリームを RMU が `DescribeStream` できず（`ResourceNotFoundException`）、リードモデルに反映されなかった。`/tmp/localstack` の bind mount を残した場合は v4 が起動時にその領域を消すため `dynamodb-setup` が exit 255 で失敗する。バージョン統一はイメージの差し替えではなく各 stack の v4 移行なので、別作業にする。go の変更は戻した。
- §7.5 のうち、1回だけ読む検証、Java の `editMessage` 未検証、Java の GraphQL パス食い違いは、現行コードでは解消済み。経緯は [acceptance-criteria.md §7.5](acceptance-criteria.md#75-現行-e2e-の差) に記す。
- 受け入れ基準の §7 以外の全項目の再監査は未実施。

### 完成条件に含めない既知の事項

Java の [Bump-Version の token 入力エラー](https://github.com/j5ik2o/cqrs-es-example-java/actions/runs/42562183966) と Scala の Lambda 配布物サイズ問題は、公開・AWS 配布に関する事項。ローカル Compose 稼働の積み残しには数えない。詳細なサイズ値と旧方針の経過は git 履歴から参照する。

### open PR と issue

GitHub の open 一覧を7 repo 全件で確認した。PR は [Rust #591](https://github.com/j5ik2o/cqrs-es-example-rs/pull/591) の1本のみ。実装系の open PR・issue はない。残る issue は次の6件。

- ハブ [#3](https://github.com/j5ik2o/cqrs-es-example/issues/3): M2 追跡。
- Rust [#551](https://github.com/j5ik2o/cqrs-es-example-rs/issues/551): Renovate 整理、[#8](https://github.com/j5ik2o/cqrs-es-example-rs/issues/8): Dependency Dashboard。
- Go [#7](https://github.com/j5ik2o/cqrs-es-example-go/issues/7)、Java [#4](https://github.com/j5ik2o/cqrs-es-example-java/issues/4)、JS [#4](https://github.com/j5ik2o/cqrs-es-example-js/issues/4): Dependency Dashboard。

Java #588 と JS #830 は open 一覧にない。Scala と Kotlin は open PR・issue ともに0。

Rust #591 の扱いでは、現行 `.github/workflows/ci.yml` が push と PR の両方で `tools/**` を `paths-ignore` に含む点に注意する。定期 CI の成功だけで同 PR の変更を検証済みとは判定しない。

## 引き継ぐ作業制約

対象範囲は冒頭の今回の指示を優先し、旧版の実 AWS 適用や ESA 分割待ちを作業候補に戻さない。

[旧版 §6](https://github.com/j5ik2o/cqrs-es-example/blob/a6abee66f69e043dac083ed6bbbde1492921e3c1/docs/HAND_OFF.md#6-加藤さんの決定禁止事項) の制約は、コードの変化だけで解除されたとは扱わない。Renovate の停止設定は go・java・js・rs の `renovate.json` の `enabled: false` でも確認できる。既存 PR のマージ履歴から停止方針の全面解除を推定しない。

- Renovate PR の変更・整理・クローズには別の指示が必要。
- AGENTS.md の編集、docker prune、他担当の資源削除、force push・rebase、ESA 本体の変更を行わない。
- common repo の [event-store-adapter #83](https://github.com/j5ik2o/event-store-adapter/pull/83) の再実装をしない。
- main に直 push せず、PR 経由とする。HAND_OFF の PR は同時に1本まで。マージ前に CI と未解決レビューを確認する。

今回の作業は、事実確認、対象範囲の文書化、6言語の Compose e2e のローカル実行。端末・worktree・担当者の古い ID は削除し、必要時に実環境から調べ直す。

## 次回の確認方法

各 repo でリモートを取得し、固定した確認コミットとの差分を読む。ローカルの HEAD が main 最新とは限らないため、他 repo の checkout や未コミット変更をそのまま使わない。

```sh
git fetch origin main
git rev-parse origin/main
git log -12 --oneline origin/main
git diff <この文書の確認コミット>..origin/main -- <対象ファイル>
git show origin/main:<対象ファイル>
```

PR・issue と CI はコードとは別に取得する。通常 CI と release・バージョン更新などの補助ワークフローを区別し、各実行の `headSha` と確認コミットを照合する。

```sh
gh pr list --repo j5ik2o/<repo> --state open --limit 100
gh issue list --repo j5ik2o/<repo> --state open --limit 100
gh run list --repo j5ik2o/<repo> --branch main --limit 20 \
  --json workflowName,headSha,status,conclusion,url
```

事実確認後に、確認日時・コミット・該当する根拠だけを置き換える。経過の追記、完了済み PR の累積一覧、旧担当者の状態はこの文書に持ち越さない。
