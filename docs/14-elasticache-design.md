# 14. ElastiCache の構成選定

> 前提: [11. メモリ管理とエビクション](11-memory.md)、[12. 永続化・バックアップ・復旧](12-persistence-and-backup.md)、[13. レプリケーションとクラスター](13-replication-and-cluster.md)

## 要点

- 最初の選択は **Serverless かノードベースか**です。Serverless は容量の管理が要らず始めやすい一方、設定の変更や調査用のコマンドに制約があります。ノードベースは細かく制御でき、トラフィックが読めるなら費用も下げやすくなります。
- ノードベースでは、**クラスターモードを有効にするか**、**ノードタイプとシャードの数**、**レプリカの数とマルチ AZ** を決めます。
- サイジングは、メモリ（11 章の見積もり）、CPU、ネットワークの3つで行います。
- Serverless の費用は、**データ量**と**リクエストの量（ECPU）**で決まります。ECPU は転送するデータの大きさにも比例するので、値の大きさが費用に効きます。
- CDK の L1 コンストラクト（`CfnServerlessCache`、`CfnReplicationGroup` など）で構成を定義します。暗号化や認証など、作った後で変えにくい設定は、最初から決めておきます。

## Serverless かノードベースか

### 違い

| 観点 | Serverless | ノードベース |
|---|---|---|
| 容量の管理 | 不要。自動で拡大・縮小する | ノードタイプとノードの数を自分で決める |
| クラスターモード | 常に有効 | 有効・無効を選べる |
| TLS・保存時の暗号化 | 常に有効 | 選べる（ただし CDK のドキュメントでは、Valkey のレプリケーショングループを新しく作るときは、転送時の暗号化が必須とされている） |
| パラメータの変更 | **できない**（エビクションのポリシーは `volatile-lru` に固定） | パラメータグループで変えられる |
| 調査用のコマンド | `KEYS`、`SLOWLOG`、`MEMORY`、`OBJECT`、`CLIENT LIST` などが使えない（17 章） | 一部（`CONFIG` など）を除き使える |
| 上限 | キャッシュ全体で 5,000 GiB・毎秒 1,500 万 ECPU。**ハッシュスロットごとに 32 GiB・毎秒 3 万 ECPU**（レプリカから読む場合は 9 万） | ノードタイプとシャードの数で決まる |
| 耐久性の機能（12 章） | 使えない | 使える（9.0 以降） |
| データ階層化（SSD を使うノード） | 使えない | 使える |
| リザーブドノード（費用の割引） | 使えない | 使える |
| 課金 | データ量（GB 時間）＋リクエストの量（ECPU） | ノードの時間 |

### 選び方

| 状況 | 向いている構成 |
|---|---|
| トラフィックがまだ読めない。まず始めたい | Serverless |
| 運用の手間を最小にしたい | Serverless |
| トラフィックが安定していて、長期で使う（リザーブドノードで費用を下げたい） | ノードベース |
| エビクションのポリシーなど、パラメータを変えたい | ノードベース |
| 障害の調査で `SLOWLOG` や `MEMORY USAGE` を使いたい | ノードベース |
| 耐久性の機能、データ階層化を使いたい | ノードベース |
| 1つのキーや1つのハッシュスロットへのアクセスが非常に多い | ノードベース（Serverless のハッシュスロットごとの上限に注意） |

この資料の想定する使い方（作り直せる読み取り用のデータストア）では、Serverless のエビクションのポリシーが `volatile-lru` に固定されていても、データに有効期限を付けなければ消されません（11 章）。その点では、Serverless でも問題はありません。
判断の分かれ目は、**運用の手間を減らすこと**と、**細かい制御・調査のしやすさ・費用の最適化**の、どちらを重く見るかです。

> **注意: Serverless では使えないコマンドがある**
>
> - **何が起きるか**: ローカルやノードベースの検証環境で動いていた調査用のスクリプトや運用の手順が、Serverless の本番では動かない。障害のときに、`SLOWLOG` で遅いコマンドを調べようとして、使えないことに気づく。
> - **なぜ起きるか**: Serverless では、`KEYS`、`SLOWLOG`、`LATENCY`、`MEMORY`、`OBJECT`、`MONITOR`、`CLIENT LIST` / `CLIENT KILL`、`FUNCTION` / `FCALL` などが使えない。
> - **どう防ぐか**: Serverless を選ぶなら、障害の調査はメトリクスとアプリ側のログで行う前提で、監視を設計する（16 章・17 章）。

## ノードベースの構成

### クラスターモード

| 観点 | クラスターモード無効 | クラスターモード有効 |
|---|---|---|
| シャードの数 | 1 | 1〜500 |
| データ量の上限 | 1つのノードのメモリまで | シャードの数 × 1つのノードのメモリ |
| 書き込みの処理能力 | 1つのプライマリまで | シャードの数に応じて増やせる |
| 複数のキーの操作 | 制約なし | 同じハッシュスロットに限る（13 章） |
| クライアント | `Redis` | `RedisCluster` |
| 接続先 | プライマリエンドポイント、リーダーエンドポイント | 設定エンドポイント |

データ量が1つのノードに収まり、将来も増えないなら、クラスターモード無効のほうが単純です。
ただし、**後からクラスターモード有効に変えるのは手間がかかります**（途中の「互換」の段階を経る必要がある）。データの増加が見込まれる場合や、Serverless に移る可能性がある場合は、最初からクラスターモード有効にして、アプリもクラスターを前提に作るほうが安全です（05 章）。

### ノードタイプ

| 系統 | 例 | 向いている用途 |
|---|---|---|
| メモリ最適化 | `cache.r7g`、`cache.r8g` | データ量が多い。この資料の想定する使い方の第一候補 |
| 汎用 | `cache.m7g` | メモリと CPU のバランスを取りたい |
| ネットワーク最適化 | `cache.c7gn` | 通信量が非常に多い |
| データ階層化 | `cache.r6gd` など | あまり使われないデータを SSD に置き、費用を下げる。検索の機能は使えない |
| バースト可能 | `cache.t4g` | 開発・検証用。**本番には使わない**（CPU の性能が持続しない。予約メモリも多く必要） |

### レプリカとマルチ AZ

| 設定 | おすすめ | 理由 |
|---|---|---|
| レプリカの数 | シャードごとに 1 つ以上（本番） | レプリカがないと、ノードの故障や再起動でデータが失われる（12 章） |
| マルチ AZ と自動フェイルオーバー | 有効 | AZ の障害でも止まらないようにする |
| レプリカを増やす | 読み取りの処理能力が足りないとき | レプリカから読む場合（13 章）。費用はレプリカの数だけ増える |

## サイジング

### 手順

| 観点 | 見積もり方 |
|---|---|
| メモリ | 11 章の見積もり（2世代分などを含む）を、1つのノードの maxmemory で割り、余裕を持たせてシャードの数を決める |
| CPU | 想定する最大のリクエストの量で、検証環境に負荷をかけ、`EngineCPUUtilization` を測る。コマンドの実行は1本のスレッドなので、vCPU の数を増やしても、コマンドの実行の処理能力はそのまま増えない（通信の処理は I/O スレッドで並列になる） |
| ネットワーク | 「1秒あたりのリクエストの数 × 1回の応答の大きさ」と、全件ロードの通信量を見積もり、ノードタイプのネットワークの帯域と比べる |

```text
例（ノードベース、クラスターモード有効）:
  11 章の見積もり（2世代分などを含む）: 60 GiB
  1つのノードの maxmemory: 約 26 GiB（cache.r7g.xlarge の場合）
  → シャードの数 = 60 ÷ 26 ≒ 2.3 → 余裕を見て 3 シャード
  → 各シャードにレプリカを 1 つ → ノードは合計 6 台
```

数値は説明のための例です。maxmemory の正確な値は、AWS のドキュメントのノードタイプごとの表で確認します。

> **注意: 平均のリクエストの量でサイジングする**
>
> アクセスの多い時間帯や、全件ロードの時間帯には、平均の何倍ものリクエストが来ます。CPU とネットワークは、**最も負荷の高い時間帯**で見積もります。全件ロードと API のアクセスが重なる時間帯も考えます（09 章）。

## コスト

単価は変わるので、ここでは料金の仕組みを中心に説明します。単価は、調査の時点の us-east-1 の参考値です。

### Serverless

| 課金の対象 | 内容 | 参考の単価（us-east-1） |
|---|---|---|
| データ量 | 保存しているデータの量を GB 時間で課金。最小は 100 MB | $0.084 / GB 時間 |
| ECPU | リクエストの量。読み書きでは、**転送するデータ 1 KB あたり 1 ECPU**。重いコマンドは、さらに多く消費する | 100 万 ECPU あたり $0.0023 |

AWS の料金ページの例では、10 GB のデータに毎秒 5 万回のリクエストで、1時間あたり約 $1.25 です。

> **注意: ECPU の見積もりが甘い**
>
> - **何が起きるか**: 想定よりも ECPU を多く消費し、費用が見積もりの何倍にもなる。
> - **なぜ起きるか**: ECPU は、リクエストの回数ではなく、転送するデータの大きさにも比例する。3.2 KB の値を読むと 3.2 ECPU になる。`HGETALL` で大きな Hash を読む、全件ロードで大量に書く、検索で多くの結果を返す、などで消費が増える。パイプラインで往復の回数を減らしても、ECPU は減らない。
> - **どう防ぐか**: 1回の応答の大きさの平均を見積もりに入れる。必要なフィールドだけを読む（`HMGET`、`RETURN`）。全件ロードの分も見積もる。検証環境で実際の ECPU の消費量を測る。

### ノードベース

| 課金の対象 | 内容 | 参考の単価（us-east-1） |
|---|---|---|
| ノード | ノードの時間（1時間未満は1時間として課金）。レプリカも1台として数える | `cache.r7g.xlarge` が $0.3496 / 時間 |
| リザーブドノード | 1年・3年の契約で割り引く | 最大で 55% 程度の割引 |
| 耐久性の機能（同期） | ノードの料金に上乗せ | 18% |

### 共通

| 課金の対象 | 内容 | 参考の単価 |
|---|---|---|
| バックアップの保存 | S3 に保存したバックアップの量 | $0.085 / GiB 月 |
| AZ をまたぐ通信（ノードベース） | アプリと Valkey が別の AZ にあると、アプリ側の通信に課金される | $0.01 / GB |

Valkey は、Redis OSS より安く設定されています（Serverless で 33%、ノードベースで 20%）。

> **注意: AZ をまたぐ通信の費用**
>
> ノードベースで、アプリと Valkey のプライマリが別の AZ にあると、すべての通信に AZ をまたぐ料金がかかります。通信量の多い使い方（大きな値を頻繁に読む、全件ロード）では、無視できない額になります。通信量を見積もり、費用に含めます。

## CDK での構成例

正式版の CDK（aws-cdk-lib）では、ElastiCache には L1 コンストラクト（`Cfn*`）だけが用意されています。以下は、必要な部分だけを示した例です。

### 共通: サブネットグループ

```python
from aws_cdk import aws_elasticache as elasticache

subnet_group = elasticache.CfnSubnetGroup(
    self,
    "ValkeySubnetGroup",
    description="Subnets for Valkey",
    subnet_ids=[s.subnet_id for s in vpc.private_subnets],
)
```

### Serverless

```python
serverless = elasticache.CfnServerlessCache(
    self,
    "CatalogCache",
    engine="valkey",
    serverless_cache_name="catalog",
    major_engine_version="9",
    subnet_ids=[s.subnet_id for s in vpc.private_subnets],
    security_group_ids=[valkey_sg.security_group_id],
    # 使用量の上限: 想定外の増加で費用が膨らむのを防ぐ
    cache_usage_limits=elasticache.CfnServerlessCache.CacheUsageLimitsProperty(
        data_storage=elasticache.CfnServerlessCache.DataStorageProperty(maximum=100, unit="GB"),
        ecpu_per_second=elasticache.CfnServerlessCache.ECPUPerSecondProperty(maximum=1_000_000),
    ),
    snapshot_retention_limit=7,  # 自動バックアップの保持日数（12 章）
    user_group_id=user_group.user_group_id,  # 認証（15 章）
)
# 接続先: serverless.attr_endpoint_address / serverless.attr_endpoint_port
```

### ノードベース（クラスターモード有効）

```python
params = elasticache.CfnParameterGroup(
    self,
    "ValkeyParams",
    cache_parameter_group_family="valkey9",
    description="Valkey 9 cluster mode",
    properties={
        "cluster-enabled": "yes",
        "maxmemory-policy": "noeviction",  # 11 章: データが黙って消えないようにする
    },
)

replication_group = elasticache.CfnReplicationGroup(
    self,
    "CatalogCluster",
    replication_group_description="Catalog read store",
    engine="valkey",
    engine_version="9.1",
    cache_node_type="cache.r7g.xlarge",
    cluster_mode="enabled",
    num_node_groups=3,  # シャードの数
    replicas_per_node_group=1,  # シャードごとのレプリカの数
    automatic_failover_enabled=True,
    multi_az_enabled=True,
    cache_subnet_group_name=subnet_group.ref,
    security_group_ids=[valkey_sg.security_group_id],
    cache_parameter_group_name=params.ref,
    transit_encryption_enabled=True,
    at_rest_encryption_enabled=True,
    snapshot_retention_limit=7,
    snapshot_window="18:00-19:00",  # UTC。全件ロードの時間帯と重ねない（11 章）
    preferred_maintenance_window="sun:19:00-sun:20:00",  # UTC（18 章）
    user_group_ids=[user_group.user_group_id],
)
# 接続先: replication_group.attr_configuration_end_point_address / ..._port
```

| 注意点 | 内容 |
|---|---|
| パラメータグループのファミリー | Valkey 9 は `valkey9`。CDK のドキュメントの一覧にまだ載っていないことがあるので、デプロイして確かめる |
| Serverless のエンジンのバージョン | `major_engine_version` に指定できる値は、CDK のドキュメントに書かれていない。上の例の `"9"` も、デプロイして確かめる |
| 既定のパラメータグループ | 既定のものは値を変えられない。変えたい値があれば、上のように自分で作る |
| 耐久性の機能 | `durability` プロパティで指定する（作った後では、有効・無効を切り替えられない）。使う場合は、作るときに決める（12 章） |
| 作った後で変えにくい設定 | 暗号化、クラスターモード、耐久性の機能、サブネット。最初に決める |

## まとめ

- Serverless は運用の手間が少なく始めやすい一方、パラメータの変更や調査用のコマンドに制約があり、ハッシュスロットごとの上限もあります。ノードベースは細かく制御でき、トラフィックが読めるなら費用を下げやすくなります。
- ノードベースでは、データの増加や Serverless への移行の可能性があるなら、最初からクラスターモード有効にします。本番では、シャードごとにレプリカを置き、マルチ AZ を有効にします。
- サイジングは、メモリ（2世代分を含む）、CPU（最も負荷の高い時間帯）、ネットワークの3つで行い、検証環境で測って確かめます。
- Serverless の ECPU は転送するデータの大きさに比例します。必要なフィールドだけを読み、全件ロードの分も見積もります。ノードベースでは、AZ をまたぐ通信の費用も考えます。
- CDK では L1 コンストラクトで定義します。暗号化、クラスターモード、耐久性の機能など、作った後で変えにくい設定は最初に決めます。
