# 全言語共通の受け入れ基準

この文書は、`j5ik2o/cqrs-es-example` と 6 つの言語別リポジトリを更新するときの、全言語共通の完成条件を定める。各言語の作業者は、着手前にこの文書を唯一の基準として読む。基準日は 2026-11-06 で、ESA のバージョンと各リポジトリの構成は、同日に各リポジトリのファイルを読んで確認した。

この文書には 2 種類の記述がある。決定済みの要件と、判断待ちの設計判断である。判断待ちの項目は §10 に集め、選択肢・制約・影響・推奨だけを書く。この文書では決めない。決定は加藤さんまたはアーキテクトが行う。

## 対象範囲の確定（2026-11-09）

ユーザー指示により、ゴールは全6言語がローカルの Docker Compose と LocalStack で稼働し、Write API、RMU、Read API を通した e2e が成功することとする。実 AWS へのデプロイ、Secrets Manager・SNS・アラーム通知先の設定、Lambda 配布物サイズへの適合、公開・リリースは完成条件に含めない。Scala のローカル稼働に ESA の成果物分割を待つ必要はない。

以下にある「現在」「現行」の実装状況は基準日 2026-11-06 の記録である。実装の SSoT は各 repo の git 履歴とコードとし、確認コミットと CI 実行の索引は [HAND_OFF.md](HAND_OFF.md) を参照する。

このゴールは §8.1 のうち「`docker compose` と LocalStack で Write、RMU、Read の順に確認する e2e が成功する」を6言語で満たすことまでを指す。§8.1 の残りの項目、とくに main と Pull Request の CI 3ジョブの必須化、README と `docs/` の整備、Renovate の整理は、このゴールの達成だけでは満たされない。§8.1 は各言語の完了条件として残る。

## 用語

| 用語 | 意味 |
|:--|:--|
| ESA | Event Store Adapter。CQRS/イベントソーシングのイベント store を言語ごとに実装したライブラリ群 |
| envelope | ESA v5 が定めるイベントとスナップショットの封筒。メタデータと payload を分けて持つ |
| journal | イベントを追記する DynamoDB テーブル |
| snapshot | 集約の状態を保存する DynamoDB テーブル。現在値と履歴を持つ |
| head | 集約ごとの最新 `seq_nr` と直近のイベントを持つ DynamoDB テーブル。変更フィードの単位 |
| RMU | Read Model Updater。イベントを読んでリードモデルを更新するアプリケーション |
| リードモデル | Read API が参照する参照用のデータ。現行実装では MySQL 上のテーブル |

## 1. 対象リポジトリと目標バージョン

対象は、ハブの `j5ik2o/cqrs-es-example` と、次の 6 つの言語別リポジトリである。目標バージョンは、ESA の v5 契約に適合する安定版である。

| リポジトリ | 言語 | 目標 ESA | 現在の ESA | 現在の状態 |
|:--|:--|:--|:--|:--|
| `cqrs-es-example-go` | Go | 5.1.0 | 1.0.197 | 実装あり。v5 未対応 |
| `cqrs-es-example-java` | Java | 5.1.0 | 1.1.174 | 実装あり。v5 未対応 |
| `cqrs-es-example-kotlin` | Kotlin | 5.1.0 | 未着手 | `README.md` のみ |
| `cqrs-es-example-scala` | Scala | 5.1.0 | 1.0.142 | ドメイン層のみ。アプリなし |
| `cqrs-es-example-js` | TypeScript | 5.2.0 | 3.1.0 | 実装あり。v5 未対応 |
| `cqrs-es-example-rs` | Rust | 5.2.0 | 1.1.34 | 実装あり。v5 未対応 |
| `cqrs-es-example` | なし | なし | なし | この文書を置くハブ。CI なし |

目標バージョンは、Orca の handoff にある計画ファイル `cqrs-es-plan.md` の指定に従う。2026-11-06 時点で、ESA の各リポジトリには `v5.1.0` と `v5.2.0` のタグが存在することを確認した。Go・Java・Kotlin・Scala は `v5.1.0`、JS・Rust は `v5.2.0` を指す。

完了の単位はマイルストーンで分かれる。M2 で rs、js、java、go を v5 に移行する。M3 で Kotlin を新規に作る。M4 で Scala を作り直す。

## 2. 共通アーキテクチャ

### 2.1 コマンド側とクエリ側を分ける

このサンプルは CQRS とイベントソーシングを組み合わせる。コマンド側はイベントを書き、クエリ側はリードモデルを読む。両者は同じデータベースを共有しない。コマンド側の記録は DynamoDB のイベントで、クエリ側が読むのは MySQL のリードモデルである。リードモデルはイベントから作る派生データであり、いつでも作り直せる。

```text
Client
  |-- mutation --> Write API --> ESA --> DynamoDB
  |                                      journal
  |                                      snapshot
  |                                      head --- Streams --> RMU --> MySQL (read model)
  |
  `-- query ----> Read API --------------------------------> MySQL (read model)
```

### 2.2 3 つのアプリケーションの責務

**Write API Server** は GraphQL の Mutation と `healthCheck` Query を公開する。入力を受け取り、ドメインのコマンドを実行し、ESA を使ってイベントとスナップショットを保存する。集約の復元は ESA に任せる。リードモデルは読まない。

**Read API Server** は GraphQL の Query だけを公開する。MySQL のリードモデルだけを読み、DynamoDB の journal、snapshot、head は読まない。`userAccountId` を受け取り、そのアカウントが閲覧できるデータだけを返す。

**Read Model Updater（RMU）** は、head テーブルの DynamoDB Streams を購読する。head の変化ごとに、その書き込みで追加されたイベントの envelope を読み、リードモデルへ反映する。同じイベントを二重に適用しても結果が変わらないようにする。`(aggregate_id, seq_nr)` で重複を捨て、`seq_nr` の飛びを見つけたら journal から埋める。処理済みの位置は `read_model_positions` に記録する。

ESA v5 では、変更フィードは head テーブルの Streams である。journal の Streams は集約ごとの順序を保証しないため、使わない。現行実装は journal の Streams を購読しており、ここが v5 移行で最も大きな変更になる。

RMU のローカル実行は必須とする。AWS Lambda 版は JS だけが持っており、共通の必須条件にはしない。

## 3. DynamoDB の 3 テーブル

### 3.1 3 テーブルが指すもの

ESA v5 の DynamoDB ストレージは、`journal`、`snapshot`、`head` の 3 テーブルを使う。Read Model はこの 3 テーブルに含まれない。現行実装のリードモデルは MySQL 上にあり、`group_chats`、`members`、`messages`、`read_model_positions` の 4 テーブルで構成する。Read Model を DynamoDB に移すかどうかは §10.k の判断待ちである。

| テーブル | 役割 | パーティションキー | ソートキー | GSI | Streams | TTL |
|:--|:--|:--|:--|:--|:--|:--|
| `journal` | イベントの追記ログ | `aid` (S) | `seq_nr` (N) | なし | 無効 | なし |
| `snapshot` | 集約状態のキャッシュと履歴 | `aid` (S) | `skey` (N)。現在値は 0、履歴は `seq_nr` | `(aid, active_history_seq_nr)`、projection は `KEYS_ONLY` | 無効 | 属性 `ttl` |
| `head` | 集約ごとの最新 `seq_nr` と直近イベント | `aid` (S) | なし | なし | 有効、`NEW_IMAGE` | なし |

仕様の根拠は ESA の [DynamoDB storage profile](https://github.com/j5ik2o/event-store-adapter/blob/main/docs/spec/storage/dynamodb.md) である。

### 3.2 キーの作り方

`aid` は `${typeName}-${value}` である。ESA の `AggregateId` がこの形に組み立てる。型名にハイフンを含めてはならない。値には含めてよい。`aid` は UTF-8 で 1024 バイト以下にする。論理シャードやハッシュによるキー変換は行わない。

3 テーブルには、同じ `store_id` と `layout_version=1` を持つ設定項目を 1 つずつ置く。キーは `aid="__config__"`、journal は `seq_nr=0`、snapshot は `skey=0`、head はソートキーなしである。ストアの構築時に 3 つの設定項目を強い整合性で読み、`store_id` と `layout_version` が一致しない場合は設定エラーにする。

### 3.3 書き込みと読み取り

1 回の書き込みは 1 つの `TransactWriteItems` で行う。journal への Put、head への Put または Update、必要なら現在の snapshot と履歴 snapshot への Put をまとめる。新しい集約は `seq_nr=1` で head が存在しないことを条件にし、更新は `head.seq_nr = event.seq_nr - 1` を条件にする。

読み取りは集約 ID を指定した強い整合性の読み取りにする。`getLatestSnapshotById` は head と現在の snapshot を `BatchGetItem` で読む。この 2 つは原子的ではない。snapshot が head より古い場合も新しい場合もあり得る。復元は snapshot の `seq_nr + 1` から始め、snapshot がなければ 1 から始める。

### 3.4 現行実装との違い

現行の 4 リポジトリは ESA v5 より前の契約で動いている。テーブルは `journal` と `snapshot` の 2 つで、どちらも `pkey` と `skey` をキーにし、`aid` と `seq_nr` の GSI を持つ。`head` テーブルはない。journal の Streams を RMU が購読している。

v5 に移行すると、テーブル定義は §3.1 の形に変わる。この文書では、既存のサンプルデータを移行せず、LocalStack のテーブルを作り直す前提に立つ。前提の確認は §10.i に置く。

物理名は設定値である。現行の `journal`、`snapshot` という名前をそのまま使うか、環境ごとの接頭辞を付けるかは §10.b と §10.d の判断待ちである。

## 4. envelope API

### 4.1 イベント envelope

ESA v5 では、イベントは envelope で運ぶ。イベント自体にメタデータを持たせない。

| 要素 | 内容 |
|:--|:--|
| `aggregate_id` | 集約 ID |
| `seq_nr` | 1 以上の整数。ドメインが採番する |
| `occurred_at` | ドメインが決めた発生時刻。ナノ秒精度 |
| `manifest` | 利用者が付ける自由文字列。省略時は空文字 |
| `payload` | ドメインの内容 |

必須規則は次のとおりである。

- `aggregate_id`、`seq_nr`、`occurred_at`、`payload` は構築時に必ず指定する。省略できるのは `manifest` だけである。
- `occurred_at` は Unix epoch からのナノ秒が符号付き 64 ビット整数に収まる範囲にする。
- `seq_nr` は 0 以上 2^53−1 以下にする。イベントは 1 から始まる。
- ライブラリは `manifest` と `payload` の中身を解釈しない。
- シリアライザが扱うのは `payload` だけである。`aggregate_id`、`seq_nr`、`manifest` を payload に埋め込まない。
- 既定のシリアライザは JSON とする。
- envelope は不変にし、将来メタデータを追加しても既存の利用者コードを壊さない形で公開する。

### 4.2 スナップショット envelope

| 要素 | 内容 |
|:--|:--|
| `aggregate` | スナップショット時点の集約状態 |
| `seq_nr` | この状態が反映した最後のイベントの `seq_nr` |
| `manifest` | 集約状態の型やスキーマ版を示す自由文字列。省略時は空文字 |

head の `seq_nr` はスナップショット envelope に含めない。読み取り時に envelope とは別に返す。

### 4.3 操作

ESA v5 の store は次の 4 操作を提供する。

| 操作 | 結果 |
|:--|:--|
| `persistEvent(event)` | イベントを 1 つ追記する |
| `persistEventAndSnapshot(event, snapshot)` | イベントとスナップショットを同時に書く |
| `getLatestSnapshotById(id)` | スナップショット envelope と head の `seq_nr` を返す |
| `getEventsByIdSinceSeqNr(id, seqNr)` | `seqNr` 以上のイベント envelope を昇順で返す |

`seq_nr=1` のイベントは集約の作成である。2 以降のイベントは直前の `seq_nr` に続かなければならない。重複や古い `seq_nr` は楽観ロックの失敗にし、飛びは契約違反にする。

### 4.4 後方互換とバージョン管理

`manifest` は自由文字列なので、イベント型名やスキーマ版を入れる余地がある。ただし、ライブラリは解釈しない。RMU がイベント型を判別する方法、payload のスキーマをどう版管理するか、未知のフィールドをどう扱うかは §10.c の判断待ちである。

現行実装は、イベントの payload に `type`、`aggregateId`、`occurredAt`、`isCreated` などのメタデータを含めている。v5 ではこれらを envelope に移す。したがって、ドメインのイベント型とシリアライザは全言語で破壊的変更になる。既存サンプルデータの移行は行わない前提であり、確認は §10.i に置く。

言語ごとの時刻型には差がある。Java は `Instant`、Go は `time.Time`、JS は `Date` を使う。JS の `Date` はミリ秒精度なので、v5 の規則に従いミリ秒に丸めることを許す。

## 5. 共通 GraphQL スキーマ（GroupChat）

### 5.1 正本と配布方法は判断待ち

共通スキーマの正本をハブに置くか、各リポジトリに複製するか、codegen で配るかは §10.a の判断待ちである。この節は、操作の集合と型の案を示す。型名とスカラー名の統一は §10.j の判断待ちであり、ここで決定しない。

### 5.2 必須の操作

共通スキーマでは、Write API に次の 8 つの Mutation と、`healthCheck` の Query を必須とする。

| Mutation | 入力 | 返す値 |
|:--|:--|:--|
| `createGroupChat` | `name`, `executorId` | `groupChatId` |
| `deleteGroupChat` | `groupChatId`, `executorId` | `groupChatId` |
| `renameGroupChat` | `groupChatId`, `name`, `executorId` | `groupChatId` |
| `addMember` | `groupChatId`, `userAccountId`, `role`, `executorId` | `groupChatId` |
| `removeMember` | `groupChatId`, `userAccountId`, `executorId` | `groupChatId` |
| `postMessage` | `groupChatId`, `content`, `executorId` | `groupChatId`, `messageId` |
| `editMessage` | `groupChatId`, `messageId`, `content`, `executorId` | `groupChatId` |
| `deleteMessage` | `groupChatId`, `messageId`, `executorId` | `groupChatId` |

`role` は `MEMBER` と `ADMIN` の列挙型とする。

共通スキーマでは、Read API に次の 6 つの Query を必須とする。

| Query | 引数 | 返す値 |
|:--|:--|:--|
| `getGroupChat` | `groupChatId`, `userAccountId` | `GroupChat` |
| `getGroupChats` | `userAccountId` | `[GroupChat!]!` |
| `getMember` | `groupChatId`, `userAccountId` | `Member` |
| `getMembers` | `groupChatId`, `userAccountId` | `[Member!]!` |
| `getMessage` | `messageId`, `userAccountId` | `Message` |
| `getMessages` | `groupChatId`, `userAccountId` | `[Message!]!` |

Subscription は Go の read スキーマにだけ `groupChats(groupChatId: String!): String!` がある。他の言語にはない。共通の必須条件に含めるかは §10.j の判断待ちである。

### 5.3 型の案

Read API の型は次の 3 つを基本にする。

```graphql
type GroupChat {
  id: String!
  name: String!
  ownerId: String!
  createdAt: NaiveDateTime!
  updatedAt: NaiveDateTime!
}

type Member {
  id: String!
  groupChatId: String!
  userAccountId: String!
  role: String!
  createdAt: NaiveDateTime!
  updatedAt: NaiveDateTime!
}

type Message {
  id: String!
  groupChatId: String!
  userAccountId: String!
  text: String!
  createdAt: NaiveDateTime!
  updatedAt: NaiveDateTime!
}
```

### 5.4 現行スキーマとの差

| 項目 | Go | Java | JS | Rust |
|:--|:--|:--|:--|:--|
| Read の型名 | `GroupChat`、`Member`、`Message` | `GroupChatOutput`、`MemberOutput`、`MessageOutput` | `GroupChatOutput`、`MemberOutput`、`MessageOutput` | `GroupChat`、`Member`、`Message` |
| Read のルート型 | `QueryRoot` | `Query` | `Query` | `QueryRoot` |
| 時刻スカラー | `NaiveDateTime` | `LocalDateTime` | `Date` | `NaiveDateTime` |
| `editMessage` | あり | なし。e2e はコメントアウト | なし | あり |
| `role` の型 | `Role` 列挙型 | `Role` 列挙型 | `String` | `MemberRole` |
| `healthCheck` の戻り値 | `String!` | `String!` | `HealthCheckOutput!` | `String!` |
| Subscription | あり | なし | なし | なし（`EmptySubscription`） |

`editMessage` は Go と Rust にあり、Java と JS にない。e2e は Java と Rust が `editMessage` を呼ぶが、Java はその部分をコメントアウトしている。共通スキーマを決めるときは、この差を埋める必要がある。

## 6. アプリケーション

### 6.1 3 つのアプリを起動する

各リポジトリは、Write API、Read API、RMU の 3 つを起動できるようにする。ローカルでは `docker compose` でまとめて起動する。設定は環境変数で渡し、エンドポイントやテーブル名をコードに埋め込まない。

| アプリ | 公開するもの | 必須の設定 |
|:--|:--|:--|
| Write API | GraphQL Mutation と `healthCheck` | journal、snapshot、head のテーブル名、snapshot の GSI 名、AWS の接続先、API の host と port |
| Read API | GraphQL Query | リードモデルの接続先、API の host と port |
| RMU | なし。Streams を購読する | head のテーブル名、Streams の接続先、リードモデルの接続先、最大取得件数 |

### 6.2 現行のポートとパス

2026-11-06 時点の値である。共通化するかどうかは §10.d と §10.e の判断待ちである。

| リポジトリ | Write API | RMU | Read API | GraphQL のパス | 起動コマンド |
|:--|:--|:--|:--|:--|:--|
| Go | 28080 | 28081 | 28082 | `/` | `writeApi`、`localRmu`、`readApi` |
| Java | 58080 | 58081 | 58082 | `/graphql` | `-w`、`-l`、`-r` |
| JS | 38080 | 38081 | 38082 | `/` | `writeApi`、`dynamodbLocalRmu`、`readApi` |
| Rust | 18080 | 18081 | 18082 | `/query` | アプリごとのバイナリ |
| Scala | なし | なし | なし | なし | なし |
| Kotlin | なし | なし | なし | なし | なし |

### 6.3 環境変数の系統

現行の環境変数は言語ごとに接頭辞が違う。

| リポジトリ | 永続化 | ストリーム | リードモデル | AWS |
|:--|:--|:--|:--|:--|
| Go | `PERSISTENCE_*` | `STREAM_*` | `DATABASE_URL` | `AWS_REGION`、`AWS_DYNAMODB_*` |
| Java | `APP_PERSISTENCE_*` | `APP_STREAM_*` | `SPRING_DATASOURCE_*` | `APP_DYNAMO_DB_*` |
| JS | `PERSISTENCE_*` | `STREAM_*` | `DATABASE_URL` | `AWS_REGION`、`AWS_DYNAMODB_*` |
| Rust | `APP__PERSISTENCE__*` | `APP__STREAM__*` | `APP__DATABASE__URL` | `APP__AWS__*` |

v5 では、少なくとも head テーブル名と snapshot の GSI 名を設定に加える。journal の GSI は不要になる。名前を全言語でそろえるかは §10.d の判断待ちである。

### 6.4 起動と停止

各リポジトリは、次の操作を README または `docs/` に書く。

- イメージをビルドするコマンド
- `docker compose` で LocalStack、MySQL、migration、3 アプリを起動するコマンド
- 停止するコマンド
- e2e を実行するコマンド
- ローカルでアプリを単体起動するコマンド

現行のスクリプト名はリポジトリごとに違う。Go、Java、Rust、JS は `tools/scripts/docker-compose-up.sh` と `tools/scripts/docker-compose-e2e-test.sh` を持つ。Go は `Makefile` の `verify-group-chat` から e2e を呼ぶ。

## 7. e2e

### 7.1 compose の構成

e2e は `docker compose` と LocalStack で動かす。最低限、次のサービスを起動する。

| サービス | 役割 |
|:--|:--|
| `localstack` | DynamoDB、DynamoDB Streams、必要なら Lambda を提供する |
| `mysql` | リードモデルを保存する |
| `migration` | リードモデルのスキーマを作る |
| `dynamodb-setup` | 3 テーブル、GSI、設定項目を作る |
| `write-api-server-1` | Write API |
| `read-model-updater-1` または `dynamodb-read-model-updater-1` | RMU |
| `read-api-server-1` | Read API |
| `e2e-test` | 検証スクリプトを実行する |

### 7.2 検証の手順

1. イメージをビルドする。
2. compose を起動する。
3. Write API と Read API の GraphQL が応答するまで待つ。`healthCheck` か `__typename` で確認する。
4. `createGroupChat` を実行し、`groupChatId` を得る。
5. `addMember` を実行する。
6. `postMessage` を実行し、`messageId` を得る。
7. リードモデルに反映されるまでポーリングする。固定の `sleep` だけで判定しない。
8. `getGroupChat`、`getGroupChats`、`getMembers`、`getMessages` を実行し、書き込んだ内容が返ることを確認する。
9. `renameGroupChat`、`editMessage`、`deleteMessage`、`removeMember`、`deleteGroupChat` を実行し、それぞれの結果を確認する。
10. どこかで GraphQL の `errors` が返ったら、スクリプトを非ゼロで終了する。

### 7.3 成功の判定

e2e の成功は、検証スクリプトが終了コード 0 で終わり、次のすべてを満たすことである。

- Write API の Mutation が `errors` を返さない。
- Read API の Query が書き込み後の値を返す。
- リードモデルの反映待ちがタイムアウトしない。
- compose のサービスが異常終了していない。

失敗時は、どのサービスがどの時点で失敗したか分かるログを出す。CI では、失敗時に compose のログを取得する。

### 7.4 CI で必須にするもの

main ブランチと Pull Request の両方で、次のジョブを必須にする。

- lint
- test
- e2e

検査を無効にしたり、失敗を無視したりして緑にしない。e2e はビルドしたイメージと LocalStack を使い、実際の Write から Read までを通す。

2026-11-09 に6 repo の main で `lint`・`test`・`e2e` を必須にした。go・java・rs は必須化の前提として CI の `paths-ignore` を外している。リポジトリごとの check 名は [HAND_OFF.md の「必須チェックの設定」](HAND_OFF.md#必須チェックの設定2026-11-09-追加) に記す。

### 7.5 現行 e2e の差

2026-11-09 に固定コミットで確認した現状は次のとおり。

- LocalStack は Java が 4.14.0、他の5言語が 2.3.2 である。Rust 2.1.0 の差は解消済み。揃えるには各 stack を v4 に移行する必要がある。go で試したところ、イメージだけの差し替えでは head テーブルのストリームを RMU が読めず e2e が落ちた。
- 6言語すべての検証スクリプトが、期限付きのポーリングでリードモデルの反映を待つ。固定の `sleep` の後に 1 回だけ読む実装は残っていない。
- Java の e2e compose は `WRITE_API_SERVER_BASE_URL` と `READ_API_SERVER_BASE_URL` を、内部サービスの URL に `/graphql` を付けて上書きする。検証スクリプトの既定値と揃っており、実行に支障はない。
- Java の e2e は `editMessage` を検証する。Java の Write スキーマに `editMessage` がある。
- JS は Lambda 版 RMU のビルドとデプロイ用スクリプトを持つが、Compose e2e はコンテナ版 RMU を使う。LocalStack 上の Lambda 実行は e2e の対象外。

## 8. 各言語の完了条件

### 8.1 共通チェックリスト

各言語のリポジトリは、次をすべて満たしたときに完了とする。

- [ ] 目標の ESA バージョンを固定し、v5 の envelope API を使っている。
- [ ] イベントのメタデータを payload に埋め込まず、envelope で運んでいる。
- [ ] journal、snapshot、head の 3 テーブルと snapshot の GSI を `dynamodb-setup` で作っている。
- [ ] 3 テーブルの設定項目 `__config__` を同じ `store_id` と `layout_version=1` で作っている。
- [ ] Write API が §5.2 の 8 つの Mutation と `healthCheck` を公開している。
- [ ] Read API が §5.2 の 6 つの Query を公開し、`userAccountId` で閲覧範囲を絞っている。
- [ ] RMU が head の Streams を購読し、`(aggregate_id, seq_nr)` で重複を捨て、飛びを journal から埋めている。
- [ ] RMU がリードモデルの処理位置を `read_model_positions` に記録している。
- [ ] `docker compose` と LocalStack で Write、RMU、Read の順に確認する e2e が成功する。
- [ ] main の CI が lint、test、e2e のすべてで緑になっている。
- [ ] Pull Request の CI も同じ 3 ジョブで緑になっている。
- [ ] 検査を無効にしたり、失敗を握りつぶしたりしていない。
- [ ] README と `docs/` に、起動方法、ポート、環境変数、e2e の実行方法を書いている。
- [ ] Renovate の Pull Request を整理し、未処理の更新を説明できる状態にしている。

### 8.2 言語ごとの範囲

| リポジトリ | 主な作業 | 完了マイルストーン |
|:--|:--|:--|
| `cqrs-es-example-rs` | ESA 5.2.0 へ移行し、head と envelope に合わせる | M2 の最初 |
| `cqrs-es-example-js` | ESA 5.2.0 へ移行し、スキーマと RMU をそろえる | M2 |
| `cqrs-es-example-java` | ESA 5.1.0 へ移行し、`editMessage` を追加する | M2 |
| `cqrs-es-example-go` | ESA 5.1.0 へ移行し、スキーマを基準に合わせる | M2 の最後 |
| `cqrs-es-example-kotlin` | Java の v5 版を骨格に、command、query と RMU、e2e と CI を作る | M3 |
| `cqrs-es-example-scala` | 他言語と同じ API、RMU、compose、CI まで作り直す | M4 |

Renovate の運用は §10.h の判断待ちである。M2 の間は ESA のメジャー更新を自動マージしない。

## 9. 言語別の現状とギャップ

### 9.1 一覧

| リポジトリ | ESA | ビルド | モジュール | 3 アプリ | compose | e2e | main CI | リードモデル |
|:--|:--|:--|:--|:--|:--|:--|:--|:--|
| Go | 1.0.197 | Go modules | command、query、rmu | あり | あり | あり | 失敗 | MySQL |
| Java | 1.1.174 | Gradle | command、query、rmu、bootstrap | あり | あり | あり | 失敗 | MySQL |
| JS | 3.1.0 | pnpm、turbo | command、query、rmu、infrastructure | あり | あり | あり | 2026-06-02 に失敗 | MySQL、Spanner 任意 |
| Rust | 1.1.34 | Cargo workspace | command、query、rmu、infrastructure | あり | あり | あり | 2026-01-05 に成功 | MySQL |
| Scala | 1.0.142 | sbt | command、query の一部、bootstrap | なし | なし | なし | なし | なし |
| Kotlin | なし | なし | なし | なし | なし | なし | なし | なし |

main CI の結果は、2026-11-06 に `gh run list` で確認できた直近の実行である。JS と Rust は、それより後の main の実行を確認できなかった。

### 9.2 Go

`go.mod` は `github.com/j5ik2o/event-store-adapter-go v1.0.197` を指す。`cmd/` に `writeApi`、`localRmu`、`readApi` があり、`pkg/command/domain`、`pkg/command/interfaceAdaptor`、`pkg/command/processor`、`pkg/query/interfaceAdaptor`、`pkg/rmu` で構成する。GraphQL は gqlgen の SDL で、Write と Read の両方に `schema.graphqls` がある。Write には `editMessage` があり、Read には `SubscriptionRoot` がある。

v5 との差は、ESA のバージョン、envelope、head テーブル、head の Streams 購読である。RMU は journal の `NEW_IMAGE` から `payload` を読み、処理位置を記録しない。e2e は `sleep 1` の後に 1 回だけ読む。CI は 2026-11-06 の main 実行が失敗している。

### 9.3 Java

`build.gradle.kts` は 5 つのモジュールで `com.github.j5ik2o:event-store-adapter-java:1.1.174` を依存に指定する。Gradle wrapper は 8.14.3、Java は 17 を対象にする。CI は Gradle 8.3 と JDK 17 を使う。`settings.gradle.kts` は `:infrastructure` を含むが、対応するディレクトリは存在しない。

GraphQL は Spring GraphQL の SDL で、`graphql-write/schema.graphqls` と `graphql-read/schema.graphqls` を持つ。Write に `editMessage` がない。Read の型名は `GroupChatOutput`、`MemberOutput`、`MessageOutput` で、スカラーは `LocalDateTime` である。RMU は AWS Lambda の `DynamodbEvent` を受け取り、旧シリアライザでイベントを復元する。処理位置の記録はない。CI は 2026-11-06 の main 実行が失敗している。

### 9.4 JS

7 つのパッケージの `package.json` が `event-store-adapter-js` 3.1.0 を依存に指定する。pnpm 9.15.9、Node 20.5 以上、turbo を使う。`packages/` は `bootstrap`、`command/{domain, interface-adaptor-if, interface-adaptor-impl, processor}`、`infrastructure`、`query/interface-adaptor`、`rmu` で構成する。

GraphQL は type-graphql の code-first で、Write に `editMessage` がない。`role` は `String`、`healthCheck` は `HealthCheckOutput` を返す。Read の型名は `GroupChatOutput`、`MemberOutput`、`MessageOutput` で、時刻は `Date` である。リードモデルは Prisma と MySQL を使い、任意で Spanner も選べる。RMU は DynamoDB Streams と Lambda の両方に対応し、処理位置を `read_model_positions` に記録する。

CI の直近の実行は 2026-06-02 で失敗している。CI は pnpm 8 を指定するが、`package.json` の `packageManager` は 9.15.9 である。

### 9.5 Rust

`Cargo.toml` の workspace 依存は `event-store-adapter-rs = "1.1.34"` である。`modules/{infrastructure, command/domain, command/processor, command/interface-adaptor-if, command/interface-adaptor-impl, query/interface-adaptor, rmu}` と、`applications/{write-api-server, read-model-updater, read-api-server}` で構成する。GraphQL は async-graphql の code-first で、Write に `edit_message` がある。Read は `EmptySubscription` を使う。AWS 向けの Terraform と Helm が `tools/deploy/` にある。

v5 との差は、ESA のバージョン、envelope、head テーブル、head の Streams 購読である。e2e は `sleep 1` の後に 1 回だけ読む。CI は 2026-01-05 の成功が直近で、それ以降の実行を確認できなかった。

### 9.6 Scala

`build.sbt` は Scala 3.4.0、sbt 1.9.9、`event-store-adapter-scala` 1.0.142 を使う。モジュールは `command-domain`、`command-interface-adaptor-if`、`command-interface-adaptor-impl`、`command-use-case`、`query-interface-adaptor`、`bootstrap` である。実装済みなのはドメイン層と、`Hello world!` を出すだけの bootstrap の入口である。Read API、RMU、3 アプリ、compose、e2e、CI はない。`README.md` は見出し 1 行だけである。

M4 で他言語と同じ形に作り直す。ESA-scala 5.1.0 は Scala 2.13.18 と 3.6.4 のクロスビルドなので、現在の 3.4.0 をどうするかは §10.f の判断待ちである。

### 9.7 Kotlin

リポジトリには `README.md` しかない。M3 で、Java の v5 版を骨格にして command、query と RMU、e2e と CI を順に作る。ESA-kotlin は 5.1.0 を目標にする。ビルド構成と JDK は §10.f の判断待ちである。

## 10. 未決の設計判断

ここに挙げる項目は、この文書では決めない。coordinator が加藤さんまたはアーキテクトに確認する。各項目は、状態、選択肢、制約、影響、推奨の順に書く。推奨は決定ではない。

### 10.a 共通 GraphQL スキーマの正本と配布方法

状態: 判断待ち

選択肢は、ハブリポジトリに SDL の正本を置く、各リポジトリに複製する、codegen で各言語の型を生成する、Git submodule や公開パッケージで配る、の 4 つである。

制約は、各言語の GraphQL ツールチェーンが違うことである。Go は gqlgen、Java は Spring GraphQL の SDL を使う。JS は type-graphql、Rust は async-graphql の code-first を使う。ハブには CI がない。

影響は、正本をハブに置くと各リポジトリのビルドがハブに依存すること、複製すると差分がドリフトすること、codegen は導入と保守のコストがかかることである。

推奨は、ハブに SDL の正本を置き、各リポジトリはビルド時に SDL を取得して自分のスキーマと比較する案である。ただし、JS と Rust が code-first である以上、生成物の一致をどう検証するかも決める必要がある。

### 10.b 3 テーブルの正確な名前・キー・GSI 設計

状態: 判断待ち

選択肢は、`journal`、`snapshot`、`head` の固定名を使う、環境ごとの接頭辞を付ける、言語ごとに分ける、の 3 つである。

キーと GSI の形は ESA v5 が定める。`journal` は `aid` と `seq_nr`、`snapshot` は `aid` と `skey` と `(aid, active_history_seq_nr)` の GSI、`head` は `aid` だけである。決めるのは物理名と GSI 名である。

制約は、同じ LocalStack で複数言語の e2e を同時に動かす可能性があること、現行の setup スクリプトが `journal` と `snapshot` の名前を前提にしていることである。

影響は、テーブル名を分けると compose と環境変数が増えること、固定名にすると同時実行で衝突することである。

推奨は、3 テーブルの論理名を固定し、物理名には環境接頭辞を付ける案である。GSI 名も設定値にし、`snapshot-active-history-index` のような役割が分かる名前にする。

### 10.c envelope のフィールドと後方互換の扱い

状態: 判断待ち

選択肢は、`manifest` にイベント型名を入れる、`manifest` にスキーマ版を入れる、両方を 1 つの文字列に詰める、payload に型の判別子を残す、の 4 つである。

制約は、ライブラリが `manifest` を解釈しないこと、RMU がイベント型を判別する必要があること、payload にはメタデータを埋め込まないことである。現行の RMU は payload の `type` を見て分岐している。

影響は、イベント型の判別方法を変えると RMU と全言語のシリアライザに影響すること、未知のフィールドの扱いを決めないとデプロイ時の互換性が崩れることである。

推奨は、`manifest` を `<event-type>@<schema-version>` の形にし、payload は純粋な JSON にする案である。未知のフィールドは読み飛ばし、必須フィールドの欠落は契約違反にする。

### 10.d DynamoDB のテーブル名・環境変数の命名規則

状態: 判断待ち

選択肢は、全言語で同じ環境変数名にする、言語ごとの接頭辞を残す、compose のサービス名から導く、の 3 つである。

制約は、現行の接頭辞が Go の `PERSISTENCE_*`、Java の `APP_*`、Rust の `APP__*`、JS の `PERSISTENCE_*` とばらばらなことである。

影響は、名前をそろえると各言語の設定クラスと compose を変更する必要があること、残すとドキュメントと運用が言語ごとに増えることである。

推奨は、テーブル名と GSI 名は共通の環境変数にし、アプリ固有の設定だけ言語ごとの接頭辞を残す案である。候補は `JOURNAL_TABLE_NAME`、`SNAPSHOT_TABLE_NAME`、`HEAD_TABLE_NAME`、`SNAPSHOT_ACTIVE_HISTORY_INDEX_NAME` である。

### 10.e LocalStack のバージョンと compose のネットワーク・ポート規約

状態: 判断待ち

選択肢は、全言語で同じ LocalStack のバージョンに固定する、言語ごとに現行のバージョンを残す、CI とローカルで分ける、の 3 つである。

制約は、LocalStack 2.x の Lambda 実行環境が Node.js のランタイムに依存すること、Docker の platform を `linux/amd64` に固定していること、現行のポートが 18080 台、28080 台、38080 台、58080 台に分かれていることである。

影響は、バージョンをそろえると Lambda の挙動が変わる可能性があること、ポートをそろえると既存のスクリプトとドキュメントを更新する必要があることである。

推奨は、LocalStack を 2.3.2 に固定し、compose のサービス名は全言語で同じにし、ホスト側のポートだけリポジトリごとに変える案である。e2e はサービス名で接続し、ホストのポートに依存しない。

### 10.f Kotlin / Scala のビルド構成と CI の JDK・Scala バージョン

状態: 判断待ち

選択肢は、Kotlin を Gradle KTS、Scala を sbt にする、両方を Gradle にする、の 2 つである。

制約は、Java が JDK 17 と Gradle 8.14.3 を使うこと、Scala の現行が Scala 3.4.0 と sbt 1.9.9 であること、ESA-scala 5.1.0 が Scala 2.13.18 と 3.6.4 のクロスビルドであること、ESA-kotlin 5.1.0 が JVM 11 を対象にしていることである。

影響は、Scala のバージョンを上げると既存のドメインコードを直す必要があること、JDK をそろえると CI の実行時間と互換性が変わることである。

推奨は、Kotlin は Gradle KTS と JDK 17（ESA の下限は 11）、Scala は sbt と Scala 3.6.4、CI の JDK は 17 にそろえる案である。Scala のクロスビルドは 3 系だけに絞る。

### 10.g 既存の AWS デプロイ用資産を残すか外すか

状態: 判断待ち

選択肢は、残して保守する、残すが対象外と明記する、削除する、別リポジトリへ移す、の 4 つである。

制約は 2 つある。Rust は `tools/deploy/` に Terraform と Helm を持ち、JS は Lambda のビルドとデプロイのスクリプトを持つ。M0 から M4 の対象はローカルの compose と LocalStack である。

影響は、残すとメンテナンス対象が増えること、削除すると AWS へのデプロイ手順を失うことである。

推奨は、当面は残し、README とこの文書で「M0 から M4 の対象外」と明記する案である。削除は、デプロイを別のマイルストーンで扱うと決めた後に行う。

### 10.h Renovate の運用

状態: 判断待ち

選択肢は、patch と minor を自動マージする、すべて手動にする、ESA だけ手動にして他は自動にする、の 3 つである。

制約は、7 つのリポジトリがあり、M2 の移行中は ESA のメジャー更新が同時に来ること、1 リポジトリあたりの WIP の Pull Request が 3 本までであることである。

影響は、自動マージを広げると CI の失敗が main に入りやすいこと、手動にすると更新が滞留することである。

推奨は、patch だけを CI 緑の条件で自動マージし、minor と major は手動にする案である。ESA はすべて手動にし、M2 の移行 PR と同じタイミングで扱う。

### 10.i ドメインイベントの互換性とデータ移行

状態: 判断待ち

選択肢は、既存データを移行しない、移行スクリプトを書く、旧テーブルを残して併用する、の 3 つである。

制約は、これはサンプル実装であり、LocalStack のデータは作り直せること、v5 のテーブル構造は旧構造と互換性がないことである。

影響は、移行しないと既存の LocalStack データは読めないこと、移行すると M2 の作業量が増えることである。

推奨は、既存データを移行せず、LocalStack のテーブルを作り直す案である。この前提が成り立つことを、着手前に確認する。

### 10.j GraphQL の型名・スカラー・操作集合

状態: 判断待ち

選択肢は、Go のスキーマを基準にする、各言語の現行名を残す、新しい共通 SDL を一から書く、の 3 つである。

制約は、Go が最も操作が多いこと、Java と JS に `editMessage` がないこと、時刻スカラーが `NaiveDateTime`、`LocalDateTime`、`Date` に分かれていること、Subscription が Go だけにあることである。

影響は、型名を変えると各言語のコード生成とクライアントに影響すること、`editMessage` を追加すると Java と JS のドメインと API を変更する必要があることである。

推奨は、Go のスキーマをたたき台にし、Read の型名を `GroupChat`、`Member`、`Message` にそろえ、時刻スカラーを `NaiveDateTime` にし、`editMessage` を必須にする案である。Subscription は必須にせず、実装する場合だけ共通スキーマに加える。

### 10.k Read Model の保存先

状態: 判断待ち

選択肢は、MySQL を続ける、DynamoDB に移す、両方を選べるようにする、の 3 つである。

制約は、現行の 4 リポジトリが MySQL を使い、JS だけが Spanner も選べること、ESA v5 の 3 テーブルにはリードモデルが含まれないことである。

影響は、DynamoDB に移すと migration と SQL が不要になる一方、クエリの書き方とローカル環境が変わることである。MySQL を続けると、3 テーブルという表現とリードモデルの保存先が別であることをドキュメントで明確にする必要がある。

推奨は、M2 から M4 では MySQL を続け、Read Model は DynamoDB の 3 テーブルとは別だと明記する案である。DynamoDB への移行は、別のマイルストーンで扱う。

## 11. 参照

- Issue: [j5ik2o/cqrs-es-example#1](https://github.com/j5ik2o/cqrs-es-example/issues/1)
- ESA v5 の共通契約: [core-contract.md](https://github.com/j5ik2o/event-store-adapter/blob/main/docs/spec/core-contract.md)
- ESA v5 の DynamoDB ストレージ: [storage/dynamodb.md](https://github.com/j5ik2o/event-store-adapter/blob/main/docs/spec/storage/dynamodb.md)
- ハブ: [j5ik2o/cqrs-es-example](https://github.com/j5ik2o/cqrs-es-example)
- Go: [j5ik2o/cqrs-es-example-go](https://github.com/j5ik2o/cqrs-es-example-go)
- Java: [j5ik2o/cqrs-es-example-java](https://github.com/j5ik2o/cqrs-es-example-java)
- JS: [j5ik2o/cqrs-es-example-js](https://github.com/j5ik2o/cqrs-es-example-js)
- Rust: [j5ik2o/cqrs-es-example-rs](https://github.com/j5ik2o/cqrs-es-example-rs)
- Scala: [j5ik2o/cqrs-es-example-scala](https://github.com/j5ik2o/cqrs-es-example-scala)
- Kotlin: [j5ik2o/cqrs-es-example-kotlin](https://github.com/j5ik2o/cqrs-es-example-kotlin)

## 変更履歴

- 2026-11-06: 初版。M0 の受け入れ基準として作成。
