# フェーズ2: 調査結果

- 作成日: 2026-11-23
- ステータス: 承認済み（2026-11-23）
- 調査時点: 2026-11-23（本文中の情報はすべてこの時点のもの）

## 0. 要約（決定事項）

| # | 論点 | 調査結果 | 決定 |
|---|---|---|---|
| D1 | 対象バージョン | OSS の最新安定版は 9.2.0（2026-11-19）。一方、ElastiCache が対応しているのは 9.1 まで | **9.1 を基準にする。9.2 は紹介しない** |
| D2 | CDK の書き方 | 正式版（aws-cdk-lib）には L1（`Cfn*`）しかない。L2 は alpha モジュールにのみある | **L1 で書く。alpha には触れない** |
| D3 | 注意の囲みの書式 | GitLab のアラート構文（`> [!warning]` など）は GitLab 17.10 以降で使える | **GitLab 固有のマイナーな構文はなるべく使わない**。囲みは通常の引用に太字のラベルを付ける形（`> **注意**`）にする。Mermaid は使う |
| D4 | 料金の書き方 | 公式の料金ページの例は us-east-1 のみ。単価は変わる | **料金体系を中心に書き、単価は参考レベルにとどめる**（調査時点の us-east-1 の値と明記する） |

## 1. Valkey 本体

### 1.1 バージョン

- 最新安定版: **9.2.0**（2026-11-19）
- サポート中のブランチ: 9.2 / 9.1（9.1.4）/ 9.0（9.0.8）/ 8.1 / 8.0 / 7.2
- 出典: https://valkey.io/download/

### 1.2 9.x 系の主な変更（本ドキュメントに関係するもの）

| 版 | 変更 | 本ドキュメントとの関係 |
|---|---|---|
| 9.0 | Hash のフィールド単位の有効期限（HEXPIRE、HSETEX、HGETEX など） | Hash の章（厚く書く） |
| 9.0 | クラスターモードで番号付き DB（SELECT）が使える | クラスターの章、キー設計の章 |
| 9.0 | アトミックなスロット移行 | クラスター・スケーリングの章 |
| 9.0 | DELIFEQ（値が一致するときだけ削除） | ロック・入れ替え処理 |
| 9.1 | 128 バイト未満の String のメモリ使用量が最大 20% 減る / Sorted Set のメモリ使用量が最大 10% 減る | メモリ設計の章 |
| 9.1 | DB 単位の ACL（`db=0,1`） | セキュリティの章 |
| 9.1 | 新コマンド: HGETDEL / MSETEX（複数キーに同じ TTL を設定）/ CLUSTERSCAN（クラスター全体をスキャン） | 一括投入、運用 |
| 9.1 | I/O スレッドの再設計（スループットが最大 17% 向上） | アーキテクチャの章（簡潔に） |
| 9.1 | Lua をモジュールとして分離（無効化できる） | スクリプトの章 |
| 9.2 | フォークを使わないスナップショット、圧縮レプリケーション、大きな Sorted Set の B+ 木化、HOTKEYS、INCREX、EXEC の条件指定、ACL ロール | **ElastiCache はまだ対応していない** → 触れる程度 |
| 9.2 | 互換性の変更: ZRANGEBYSCORE などのスコア境界に `(` 単体や空文字を渡すとエラーになる | 将来のバージョンアップ時の注意として記載 |

出典: https://valkey.io/blog/introducing-valkey-9/ 、https://valkey.io/blog/valkey-9-1-delivers-improvements-in-security-performance-and-more/ 、https://valkey.io/blog/valkey-9.2-release/

## 2. Amazon ElastiCache for Valkey

### 2.1 対応バージョン

- 7.2 / 8.0 / 8.1 / 8.2 / 9.0 / **9.1（最新）**
- ElastiCache の 8.2 は、OSS の Valkey 8.1 と互換で、ベクトル検索が追加されている
- Serverless は Valkey 7.2 以降に対応している
- Valkey のサポート終了日は公式ページに載っていない（Redis OSS 4〜6 の分だけが載っている）
- 出典: https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/engine-versions.html

### 2.2 Serverless とノードベースの違い（構成選定の章の材料）

| 観点 | Serverless | ノードベース |
|---|---|---|
| クラスターモード | **有効のみ**（クライアントがクラスターモードに対応している必要がある） | 有効 / 無効を選べる |
| TLS・保存時の暗号化 | **常に有効**（クライアントが TLS に対応している必要がある） | 選べる |
| スケーリング | 自動（容量の管理は不要） | 手動。ノードタイプとノード数を指定する |
| エンドポイント | 1つ。トポロジーが変わっても接続が切れない | 構成によって異なる |
| マルチ AZ | 3 つの AZ に非同期でレプリケーション。SLA 99.99% | シングル AZ / マルチ AZ を選べる |
| パラメータグループ | **使えない**。設定できるのはサブネット、バックアップ、使用量の上限程度 | 使える（maxmemory-policy などを細かく調整できる） |
| Serverless で使えない機能 | データ階層化、Global Datastore、リザーブドノード、耐久性（トランザクションログ） | すべて使える |
| 9.0 以降の耐久性 | — | マルチ AZ のトランザクションログ。同期書き込みはノード料金に 18% 上乗せ、非同期は追加料金なし |

出典: https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/WhatIs.deployment.html

### 2.3 料金体系

- **Serverless**
  - データ保存量（GB 時間）とリクエスト量（ECPU）で課金される
  - 読み書きでは、転送 1 KB あたり 1 ECPU。重いコマンドほど多く消費する
  - Valkey の最小課金容量は 100 MB
  - 参考値（us-east-1）: $0.084/GB 時間、100 万 ECPU あたり $0.0023
- **ノードベース**
  - ノード時間で課金される
  - リザーブドノードを使うと最大 55% 割引になる
  - 参考値（us-east-1）: cache.r7g.xlarge が $0.3496/時間
- **共通**
  - Valkey は Redis OSS より安い。Serverless で 33%、ノードベースで 20%
  - バックアップ: $0.085/GiB 月
  - ノードベースでは AZ をまたぐ通信が $0.01/GB かかる（アプリ側）
- 出典: https://aws.amazon.com/elasticache/pricing/

### 2.4 OSS の Valkey との違い（はまりポイントの材料として重要）

| 区分 | 使えないコマンド・挙動の違い |
|---|---|
| 全構成で使えない | CONFIG、BGSAVE / SAVE、DEBUG、SHUTDOWN、REPLICAOF、MIGRATE、ACL SETUSER / DELUSER（ユーザー管理は ElastiCache の API で行う）、CLUSTER のトポロジー変更コマンド |
| Serverless だけで使えない | **KEYS、SLOWLOG、LATENCY、MEMORY、OBJECT、MONITOR、CLIENT LIST / KILL など、FUNCTION / FCALL**、WAIT、SWAPDB、PSUBSCRIBE |
| Serverless の挙動 | CLUSTER コマンドからは仮想的な 1 シャードに見える。FLUSHALL / FLUSHDB は全体を消す。SCAN を多用するとリシャーディングが遅くなる |

→ Serverless では、トラブル対応でよく使うコマンド（SLOWLOG、MEMORY USAGE など）が使えない。そのため、監視・トラブルシューティングの章は構成ごとに書き分ける必要がある。

出典: https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/SupportedCommands.html

### 2.5 拡張機能の対応

| 機能 | ElastiCache での対応 |
|---|---|
| JSON | Valkey 7.2 以降 |
| 検索（属性・全文・集計） | **9.0 以降**（Serverless とノードベースの両方）。8.2 はベクトル検索のみ |
| 検索の制約 | FT.CREATE と FT.DROPINDEX はトランザクションや Lua の中で使えない。クラスターモードでは FT.SEARCH もトランザクションの中で使えない |
| Bloom | 8.1 以降（本ドキュメントでは「使わない」区分） |

出典: https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/search-features-limits.md 、https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/json-gs.md

## 3. redis-py

- 最新版: **8.1.0**（2026-07-30）。Python 3.10 以降が必要
- 非同期 API: `redis.asyncio`。クラスター向けは `redis.asyncio.cluster.RedisCluster`
- 公式にサポートしているのは Redis 7.2 以降。**Valkey のサポートは明記されていない**
  - Valkey はプロトコル互換で、Redis 7.2 系の機能を備えているので、基本的なコマンドは問題なく使える見込み
- 注意点（はまりポイントの材料）
  1. **バージョン 8.0 から RESP3 がデフォルトになった。** Valkey は RESP3 に対応しているので、基本的には問題ない。ただし、JSON・検索モジュールの応答をクライアント側で解析する部分は Redis のモジュールを前提に作られている。そのため、valkey-json と valkey-search との組み合わせは、フェーズ4で動作確認が必要
  2. **Valkey 独自のコマンドには専用のメソッドがない**（DELIFEQ、MSETEX、CLUSTERSCAN など）。`execute_command` で呼び出す必要がある
  3. **Redis 専用のコマンドが含まれている**（例: Redis 8.10 の HIMPORT）。補完候補に出てきても、Valkey では使えない
  4. **Serverless では、クラスター対応のクライアント（RedisCluster）と TLS が必須。** 単一ノード用のクライアントで接続すると問題が起きる
- 出典: https://pypi.org/project/redis/ 、https://github.com/redis/redis-py

## 4. AWS CDK

- 正式版（aws-cdk-lib）: **L1 のみ**。CfnServerlessCache、CfnReplicationGroup、CfnParameterGroup、CfnSubnetGroup、CfnUser、CfnUserGroup など
- alpha モジュール（aws-cdk.aws-elasticache-alpha、2.278.0a0）: ServerlessCache、User、UserGroup の L2 がある。ただし API は experimental。ノードベース（ReplicationGroup）の L2 はない
- 出典: https://docs.aws.amazon.com/cdk/api/v2/python/aws_cdk.aws_elasticache.html 、https://pypi.org/project/aws-cdk.aws-elasticache-alpha/

## 5. GitLab の Markdown

- アラート: `> [!note]` / `[!tip]` / `[!important]` / `[!caution]` / `[!warning]`。**GitLab 17.10 で導入された**
- 見出しを変える書き方: `> [!warning] データが消える`
- Mermaid: ` ```mermaid ` のコードブロックで描画される
- 出典: https://docs.gitlab.com/user/markdown/

## 6. 未確認事項（フェーズ5で整理）

### 解決したもの

| 事項 | 結果 | 反映先 |
|---|---|---|
| Serverless の上限 | 1キャッシュあたり 5,000 GiB・毎秒 1,500 万 ECPU。ハッシュスロットごとに 32 GiB・毎秒 3 万 ECPU（レプリカから読む場合は 9 万） | 11 章・14 章 |
| Serverless で複数の DB を使えるか | Valkey 9.0 以降、最大 1,024 個 | 02 章 |
| redis-py のクラスター用クライアントで DB 0 以外を使えるか | 使えない（例外を出す） | 08 章 |
| 再起動したときのデータ | 利用者がプライマリを再起動すると、レプリカがあってもデータが消える | 12 章・18 章 |
| フェイルオーバーにかかる時間 | AWS のドキュメントでは、昇格は通常数秒 | 12 章 |
| Serverless でレプリカから読む方法 | 読み取り用のポート（6380）と `READONLY` | 13 章 |

### 未解決のもの（本文では「確かめる」と明記している）

| 事項 | 記載している章 |
|---|---|
| redis-py（RESP3 / `protocol=2`）と valkey-search・valkey-json の組み合わせの実機での動作 | 07 章 |
| valkey-search の TEXT が日本語を単語に区切れるか | 07 章 |
| ElastiCache 9.0 / 9.1 の valkey-search で使えるオプションと、GitHub の最新ドキュメントとの差 | 07 章 |
| `RedisCluster` で Valkey 独自のコマンドを `execute_command` で呼んだときに、正しいノードに送られるか | 08 章 |
| 非同期版の `RedisCluster` で、全プライマリに対して `SCAN` する方法 | 08 章・09 章 |
| Valkey 9 系の既定のパラメータグループの `maxmemory-policy` | 11 章 |
| ノードベースでバージョンアップしたときのアプリへの影響（順番に更新されるか、フェイルオーバーが起きるか） | 18 章 |
| CDK のパラメータグループのファミリー `valkey9` と、Serverless の `major_engine_version` の値 | 14 章 |
| `default` ユーザーの扱い（AWS と CDK のドキュメントの食い違い） | 15 章 |
| ElastiCache Serverless で 9.1 が使えるか | 調査の時点で明記なし（00 章の対象バージョンは 9.1） |
| ElastiCache における Valkey のサポート終了の方針、東京リージョンの単価 | 本文では扱っていない |
