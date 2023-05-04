# Temporal Interfaces[<img src="https://img.shields.io/badge/NGSI-LD-d6604d.svg" width="90"  align="left" />](https://www.etsi.org/deliver/etsi_gs/CIM/001_099/009/01.06.01_60/gs_CIM009v010601p.pdf)[<img src="https://fiware.github.io/tutorials.Short-Term-History/img/fiware.png" align="left" width="162">](https://www.fiware.org/)<br/>

[![FIWARE Core Context Management](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/core.svg)](https://github.com/FIWARE/catalogue/blob/master/core/README.md)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.Short-Term-History.svg)](https://opensource.org/licenses/MIT)
[![Support badge](https://img.shields.io/badge/tag-fiware-orange.svg?logo=stackoverflow)](https://stackoverflow.com/questions/tagged/fiware)
[![JSON LD](https://img.shields.io/badge/JSON--LD-1.1-f06f38.svg)](https://w3c.github.io/json-ld-syntax/) <br/>
[![Documentation](https://img.shields.io/readthedocs/ngsi-ld-tutorials.svg)](https://ngsi-ld-tutorials.rtfd.io)

このチュートリアルは、Context broker 実装への**オプション**のアドオンである NGSI-LD のテンポラル・インターフェイス
の (Temporal Interfaces) 概要です。このチュートリアルでは、[以前のチュートリアル](https://github.com/FIWARE/tutorials.IoT-Agent/tree/NGSI-LD)
で接続された IoT の動物の首輪 (IoT animal collars) をアクティブにし、それらのセンサからの測定値をデータベースに保持し、
そのデータの時間ベースの集計を取得します。

チュートリアルでは全体で [cUrl](https://ec.haxx.se/) コマンドを使用しますが、
[Postman ドキュメント](https://fiware.github.io/tutorials.Short-Term-History/ngsi-ld.html) としても利用できます。

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/4824d3171f823935dcab)
[![Open in Gitpod](https://gitpod.io/button/open-in-gitpod.svg)](https://gitpod.io/#https://github.com/FIWARE/tutorials.Short-Term-History/tree/NGSI-LD)

## コンテンツ

<details>
<summary><strong>詳細</strong></summary>

-   [テンポラル・データのクエリ](#querying-temporal-data)
-   [アーキテクチャ](#architecture)
-   [前提条件](#prerequisites)
    -   [Docker と Docker Compose](#docker-and-docker-compose)
    -   [Cygwin for Windows](#cygwin-for-windows)
-   [起動](#start-up)
-   [テンポラル・オペレーションのための Orion と Mintaka の構成](#configuring-orion-and-mintaka-for-temporal-operations)
    -   [Mintaka の構成](#mintaka---configuration)
    -   [Orion の構成](#orion-configuration)
    -   [起動](#start-up-1)
        -   [Mintaka - サービスの健全性チェック](#mintaka-checking-service-health)
        -   [コンテキスト・データの生成](#generating-context-data)
    -   [テンポラル操作](#temporal-operations)
        -   [エンティティの最後の N 個のサンプル値を一覧表示](#list-the-last-n-sampled-values-for-an-entity)
        -   [エンティティの属性の最後の N 個のサンプル値を一覧表示](#list-the-last-n-sampled-values-of-an-attribute-of-an-entity)
        -   [エンティティの簡略化されたテンポラル表現](#simplified-temporal-representation-of-an-entity)
        -   [observedAt なしのテンポラル・クエリ](#temporal-queries-without-observedat)
        -   [ページネーション操作](#pagination-operations)
        -   [`q` パラメータを使用した一時的なリクエストのフィルタリング](#filtering-temporal-requests-using-the-q-parameter)
        -   [`georel` パラメータを使用したテンポラル・リクエストのジオフェンス](#geofencing-temporal-requests-using-the-georel-parameter)
-   [次のステップ](#next-steps)

</details>

<a name="querying-temporal-data"></a>

# テンポラル・データのクエリ

> "I could dance with you till the cows come home. Better still, I'll dance with the cows and you come home."
>
> — Groucho Marx (Duck Soup)

NGSI-LD は、履歴コンテキスト・データを永続化および取得するための標準化されたメカニズムを導入しています。従来、Context
Broker は現在のコンテキストのみを処理します。メモリはありませんが、NGSI-LD Context Broker を拡張して、さまざまな JSON
ベースの形式で履歴コンテキスト・データを提供できます。追加機能にはコストがかかり、パフォーマンス上の理由からデフォルト
では必須ではないため、テンポラル機能は NGSI-LD Context Broker のオプションのインターフェイスとして分類されます。

テンポラル・インターフェイスが有効になっている Context Broker は、選択したデータベースを使用して履歴コンテキストを保持
できます。NGSI-LD テンポラル・インターフェイスは、Context Broker が使用する実際の永続性メカニズムに依存しません。
インターフェイスは、さまざまなクエリが発生したときに必要な出力を指定するだけです。さらに、NGSI-LD は、`instanceId`
属性を使用して履歴コンテキストの値を修正するためのメカニズムも指定します。

結果は、`observedAt` _property-of-a-property_ を使用してタイム・スタンプが付けられた一連のデータポイントです。
タイム・スタンプが付けられた各データ・ポイントは、特定の時点でのコンテキスト・エンティティの状態を表します。個々の
データ・ポイントは、それ自体では特に有用ではありません。最大値、最小値、傾向などの意味のある統計を観察できるのは、
一連のデータポイントを組み合わせることによってのみです。

傾向データの作成と分析は、コンテキスト駆動型システムの一般的な要件です。FIWARE 内では、2つの一般的なパラダイムが使用
されています。テンポラル・インターフェイスをアクティブ化するか、個々のコンテキスト・エンティティにサブスクライブし、
それらを時系列データベースに永続化します (QuantumLeap などのコンポーネントを使用)。
後者は[別のチュートリアル](https://github.com/FIWARE/tutorials.IoT-Agent/tree/NGSI-LD)で説明しています。

このようなシステムを設計するときは、どのメカニズムを使用するかを念頭に置く必要があります。サブスクリプション・
メカニズムを使用する利点は、サブスクライブされたエンティティのみが永続化され、ディスク・スペースを節約できることです。
テンポラル・インターフェイスの利点は、Context Broker によって直接提供されることです。サブスクリプションは不要で、HTTP
トラフィックが削減されます。さらに、テンポラル・インターフェイスは、サブスクリプションを満たすエンティティだけでなく、
すべてのコンテキスト・エンティティにわたってクエリを実行できます。

#### デバイス・モニタ

このチュートリアルの目的のために、一連のダミーの動物の首輪 の IoT デバイスが作成され、Context Broker に接続されます。
使用されるアーキテクチャとプロトコルの詳細は、 IoT センサのチュートリアルに記載されています。各デバイスの状態は、
次の場所にある UltraLight デバイス・モニタの Web ページで確認できます: `http://localhost:3000/device/monitor`

![FIWARE Monitor](https://fiware.github.io/tutorials.Time-Series-Data/img/farm-devices.png)

<a name="architecture"></a>

# アーキテクチャ

このアプリケーションは、[以前のチュートリアル](https://github.com/FIWARE/tutorials.IoT-Agent/tree/NGSI-LD)で作成された
コンポーネントとダミーの IoT デバイスに基づいて構築されています。これは、2つのFIWAREのコンポーネントを使用します。
[Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) と [IoT Agent for Ultralight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/)
です。さらに、オプションのテンポラル・インターフェイスは、**Mintaka** と呼ばれるアドオンを使用してサービスされます。

したがって、アーキテクチャ全体は次の要素で構成されます:

-   [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) は、
    [NGSI-LD](https://forge.etsi.org/swagger/ui/?url=https://forge.etsi.org/rep/NGSI-LD/NGSI-LD/raw/master/spec/updated/generated/full_api.json)
    を使用して、リクエストを受信します
-   FIWARE [IoT Agent for UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/) は、
    [UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual)
    構文を使用して、デバイスから測定値を受け取り、 
    [NGSI-LD](https://forge.etsi.org/swagger/ui/?url=https://forge.etsi.org/rep/NGSI-LD/NGSI-LD/raw/master/spec/updated/generated/full_api.json)
    に変換します
-   基礎となる [MongoDB](https://www.mongodb.com/) データベース :
    -   **Orion Context Broker** が、データエンティティ、サブスクリプション、レジストレーションなどのコンテキスト・
        データ情報を保持するために使用します
    -   デバイスの URL やキーなどのデバイス情報を保持するために **IoT Agent** によって使用されます
-   [Timescale](https://www.timescale.com/) は、履歴コンテキストを永続化する時系列データベースです
-   **Mintaka** アドオン は、テンポラル・インターフェースを提供し、コンテキストを永続化します
-   **チュートリアル・アプリケーション** 次のことを行います:
    -   HTTP 上で実行される [UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual)
        プロトコルを使用して、ダミーの [農業用 IoT デバイス](https://github.com/FIWARE/tutorials.IoT-Sensors/tree/NGSI-LD)
        のセットとして機能します
-   HTTP **Web-Server** は、システム内のコンテキスト・エンティティを定義する静的な `@context` ファイルを提供します


要素間のすべての相互作用は HTTP リクエストによって開始されるため、エンティティをコンテナ化して、公開されたポートから
実行できます。

<a name="prerequisites"></a>

# 前提条件

<a name="docker-and-docker-compose"></a>

## Docker と Docker Compose

物事を単純にするために、すべてのコンポーネントが [Docker](https://www.docker.com) を使用して実行されます。**Docker**
は、さまざまコンポーネントをそれぞれの環境に分離することを可能にするコンテナ・テクノロジです。

-   Docker Windows にインストールするには、[こちら](https://docs.docker.com/docker-for-windows/)の手順に従ってください
-   Docker Mac にインストールするには、[こちら](https://docs.docker.com/docker-for-mac/)の手順に従ってください
-   Docker Linux にインストールするには、[こちら](https://docs.docker.com/install/)の手順に従ってください

**Docker Compose** は、マルチコンテナ Docker アプリケーションを定義して実行するためのツールです。
[YAML file](https://github.com/FIWARE/tutorials.Short-Term-History/blob/NGSI-LD/docker-compose/orion-ld.yml)
ファイルは、アプリケーションのために必要なサービスを構成するために使用します。つまり、すべてのコンテナ・サービスは
1 つのコマンドで呼び出すことができます。Docker Compose は、デフォルトで Docker for Windows と Docker for Mac の一部と
してインストールされますが、Linux ユーザは[ここ](https://docs.docker.com/compose/install/)に記載されている手順に従う
必要があります。

次のコマンドを使用して、現在の **Docker** バージョンと **Docker Compose** バージョンを確認できます :

```console
docker-compose -v
docker version
```

Docker バージョン 18.03 以降と Docker Compose 1.21 以上を使用していることを確認し、必要に応じてアップグレードして
ください。

<a name="cygwin-for-windows"></a>

## Cygwin for Windows

シンプルな bash スクリプトを使用してサービスを開始します。Windows ユーザは [cygwin](http://www.cygwin.com/) を
ダウンロードして、Windows 上の Linux ディストリビューションと同様のコマンドライン機能を提供する必要があります。

# 起動

開始する前に、必要な Docker イメージをローカルで取得または構築しておく必要があります。リポジトリを複製し、以下の
コマンドを実行して必要なイメージを作成してください :

```console
git clone https://github.com/FIWARE/tutorials.Short-Term-History.git
cd tutorials.Short-Term-History
git checkout NGSI-LD

./services create
```

その後、リポジトリ内で提供される
[services](https://github.com/FIWARE/tutorials.Short-Term-History/blob/NGSI-LD/services)
の Bash スクリプトを実行することによって、コマンドラインからすべてのサービスを初期化できます :

```console
./services orion|scorpio
```

> :information_source: **注:** クリーンアップをやり直したい場合は、次のコマンドを使用して再起動することができます:
>
> ```console
> ./services stop
> ```

<a name="configuring-orion-and-mintaka-for-temporal-operations"></a>

# テンポラル・オペレーションのための Orion と Mintaka の構成

スマート・ファーム内では、動物の状態に関するコンテキスト・データがさまざまなデバイスを介して受信されます。したがって、
IoT Agent を使用してデータを NGSI-LD 形式に変換します。その後、これは Context Broker で受信されます。通常、Context
Broker はシステムの最新の状態 (Mongo-DB 内) のみを保持しますが、一時的に有効な Context Broker を使用すると、Orion
はデータをタイム・スケール・データベースに保持します。この場合、Orion はタイム・スケール・データベースへのデータの
書き込みのみを担当します。これにより、システムの高速性と応答性が維持されます。Mintaka コンポーネントは、テンポラル・
インターフェース要求をリッスンし、Timescale に対して実行する関連クエリを構築する役割を果たします。全体的な
アーキテクチャを以下に示します:

![](https://fiware.github.io/tutorials.Short-Term-History/img/architecture-ld.png)

<a name="mintaka---configuration"></a>

## Mintaka の構成

```yaml
mintaka:
    image: quay.io/fiware/mintaka:${MINTAKA_VERSION}
    hostname: mintaka
    container_name: fiware-mintaka
    depends_on:
        - timescale-db
    environment:
        - DATASOURCES_DEFAULT_HOST=timescale-db
        - DATASOURCES_DEFAULT_USERNAME=orion
        - DATASOURCES_DEFAULT_PASSWORD=orion
        - DATASOURCES_DEFAULT_DATABSE=orion
        - DATASOURCES_DEFAULT_MAXIMUM_POOL_SIZE=2
    expose:
        - "8080"
    ports:
        - "8080:8080"
```

`mintaka` コンテナは、1つのポートで待機しています:

-   サービスがリッスンしているポート `8080` にテンポラル操作をリクエストする必要があります

`mintaka` コンテナは、次の環境変数によって駆動されます:

| キー                                  | 値             | 説明                                                                                         |
| ------------------------------------- | -------------- | -------------------------------------------------------------------------------------------- |
| DATASOURCES_DEFAULT_HOST              | `timescale-db` | Timescale データベースがホストされているアドレス                                             |
| DATASOURCES_DEFAULT_USERNAME          | `orion`        | Timescale データベースにアクセスするときにログインするユーザ                                 |
| DATASOURCES_DEFAULT_PASSWORD          | `orion`        | パスワードが提供されていない場合に使用するデフォルト・パスワード                             |
| DATASOURCES_DEFAULT_DATABASE          | `orion`        | `NGSILD-Tenant` ヘッダが永続コンテキストで使用されていない場合に使用されるデータベースの名前 |
| DATASOURCES_DEFAULT_MAXIMUM_POOL_SIZE | `2`            | 同時リクエストの最大数                                                                       |

<a name="orion-configuration"></a>

## Orion の構成

```yaml
orion:
    image: quay.io/fiware/orion-ld:${ORION_LD_VERSION}
    hostname: orion
    container_name: fiware-orion
    depends_on:
        - mongo-db
    ports:
        - "1026:1026"
    environment:
        - ORIONLD_TROE=TRUE
        - ORIONLD_TROE_USER=orion
        - ORIONLD_TROE_PWD=orion
        - ORIONLD_TROE_HOST=timescale-db
        - ORIONLD_MONGO_HOST=mongo-db
        - ORIONLD_MULTI_SERVICE=TRUE
        - ORIONLD_DISABLE_FILE_LOG=TRUE
    command: -dbhost mongo-db -logLevel ERROR -troePoolSize 10 -forwarding
```

`orion` コンテナは、その標準ポート `1026` で待機し、`troePoolSize` コマンド内のフラグが使用への同時接続数を制限します。

| キー                     | 値             | 説明                                                             |
| ------------------------ | -------------- | ---------------------------------------------------------------- |
| ORIONLD_TROE             | `TRUE`         | エンティティのテンポラル表現を提供するかどうか                   |
| ORIONLD_TROE_USER        | `orion`        | Timescale データベースにアクセスするときにログインするユーザ     |
| ORIONLD_TROE_PWD         | `orion`        | パスワードが提供されていない場合に使用するデフォルト・パスワード |
| ORIONLD_TROE_HOST        | `timescale-db` | Timescale データベースがホストされているアドレス                 |
| ORIONLD_MULTI_SERVICE    | `TRUE`         | マルチテナンシーを有効にするかどうか                             |
| ORIONLD_DISABLE_FILE_LOG | `TRUE`         | 速度を上げるためにファイル・ログは無効になっています             |

<a name="start-up-1"></a>

## 起動

システムを起動するには、次のコマンドを実行します:

```console
./services start
```

<a name="mintaka-checking-service-health"></a>

### Mintaka - サービスの健全性チェック

Mintaka が実行されたら、公開されたポートの `info` エンドポイントに HTTP リクエストを送信することで、ステータスを確認
できます。構成には `ENDPOINTS_INFO_ENABLED=true`と `ENDPOINTS_INFO_SENSITIVE=false` が含まれているため、エンドポイント
はレスポンスを返す必要があります。

#### :one: リクエスト:

```console
curl -L -X GET \
  'http://localhost:8080/info'
```

#### レスポンス:

レスポンスは次のようになります:

```json
{
    "git": {
        "revision": "e2352fc668dd8fc8ed340bc15447acc3c9ad40be"
    },
    "build": {
        "time": "15 September 2021, 13:48:25 +0000"
    },
    "project": {
        "artifact-id": "mintaka",
        "group-id": "org.fiware",
        "version": "0.3.26"
    }
}
```

> **トラブルシューティング:**: レスポンスがブランクの場合はどうなりますか？
>
> -   Docker コンテナが実行されていることを確認するには、
>
> ```bash
> docker ps
> ```
>
> いくつかのコンテナが実行されているのが見えるはずです。`orion` または `mintaka` 実行されていない場合は、
> 必要に応じてコンテナを再起動できます。

<a name="generating-context-data"></a>

### コンテキスト・データの生成

このチュートリアルでは、コンテキストが定期的に更新されているシステムを監視する必要があります。ダミーの IoT センサを
使用してこれを行うことができます。

農場周辺のさまざまな建物の詳細は、チュートリアル・アプリケーションにあります。`http://localhost:3000/app/farm/urn:ngsi-ld:Building:farm001`
を開いて、関連する充填センサ (filling sensor) とサーモスタットを備えた建物を表示します。

![](https://fiware.github.io/tutorials.Subscriptions/img/fmis.png)

納屋からいくつかの干し草を外し、サーモスタットを更新し、`http://localhost:3000/device/monitor` でデバイス・モニタ・
ページを開くて、**Tractor** を起動し、**Smart Lamp** をオンにします。これは、ドロップ・ダウン・リストから適切な
コマンドを選択して `send` ボタンを押すことで実行できます。デバイスからの測定値の流れは、同じページに表示されます。

<a name="temporal-operations"></a>

## テンポラル操作

システムが稼働すると、コンテキスト・データが自動的に更新され、`/temporal/entities/` エンドポイントを使用して履歴情報を
クエリできます。クエリするポートは、使用する Context Broker によって異なります。Scorpioの `/temporal/entities/`は、標準
9090 ポートに統合されています。Orion + Mintaka の場合、`/entities` は、ポート `1026` で、`/temporal/entities/` は、ポート
`8000` でリクエストされます。もちろん、これらのポートマッピングは、docker-compose ファイルを修正することで変更できます。

<a name="list-the-last-n-sampled-values-for-an-entity"></a>

### エンティティの最後の N 個のサンプル値を一覧表示

この例は、エンティティ `urn:ngsi-ld:Animal:cow002` からの最後の3つの変更を示しています

コンテキスト・エンティティ属性のテンポラル・データを取得するには、`../temporal/entities/<entity-id>` に GET リクエストを
送信します。

この `lastN` パラメータは、結果を N 個に制限します。

#### :one: リクエスト:

```console
curl -G -X GET 'http://localhost:8080/temporal/entities/urn:ngsi-ld:Animal:cow002' \
  -H 'NGSILD-Tenant: openiot' \
  -H 'Link: <http://context/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
  -d 'lastN=3'
```

単一のエンティティを要求する場合、`timerel=before` および `timeAt=<current_time>` は、テンポラル・インターフェイスの
Mintaka 実装によって暗示されるため、他の Context Broker と連携するときに、これらのパラメータを明示的に追加する必要が
ある場合があることに注意してください。

#### レスポンス:

レスポンスは単一のエンティティです。`heartRate` のように属性が変更されている場合 、最大 N 個の値が返されます。リクエストは、
プロパティのプロパティ (_properties of properties_) を含むすべての値に対して完全に正規化された JSON-LD を返すため、
どのデバイスがさまざまな読み取り値を提供し、どの単位が使用されているかを確認できます。すべての値には、サポートされている
場合は、個々のエントリをさらに操作するために使用できる `instanceId` が関連付けられています。

以下の例で、`heartRate` 及び `location` は、単一のデバイスによって提供されています。

```json
{
    "id": "urn:ngsi-ld:Animal:cow002",
    "type": "Animal",
    "heartRate": [
        {
            "type": "Property",
            "value": 51.0,
            "observedAt": "2021-09-15T13:41:37.888Z",
            "instanceId": "urn:ngsi-ld:attribute:instance:a6d2045e-162a-11ec-93ed-0242ac120106",
            "unitCode": "5K",
            "providedBy": {
                "object": "urn:ngsi-ld:Device:cow002",
                "type": "Relationship",
                "instanceId": "urn:ngsi-ld:attribute:instance:a6d20a6c-162a-11ec-93ed-0242ac120106"
            }
        },
        {
            "type": "Property",
            "value": 50.0,
            "observedAt": "2021-09-15T13:41:34.568Z",
            "instanceId": "urn:ngsi-ld:attribute:instance:a4b64b62-162a-11ec-9bda-0242ac120106",
            "unitCode": "5K",
            "providedBy": {
                "object": "urn:ngsi-ld:Device:cow002",
                "type": "Relationship",
                "instanceId": "urn:ngsi-ld:attribute:instance:a4b64d4c-162a-11ec-9bda-0242ac120106"
            }
        },
        {
            "type": "Property",
            "value": 51.0,
            "observedAt": "2021-09-15T13:41:33.508Z",
            "instanceId": "urn:ngsi-ld:attribute:instance:a43b6852-162a-11ec-9dd5-0242ac120106",
            "unitCode": "5K",
            "providedBy": {
                "object": "urn:ngsi-ld:Device:cow002",
                "type": "Relationship",
                "instanceId": "urn:ngsi-ld:attribute:instance:a43b6ba4-162a-11ec-9dd5-0242ac120106"
            }
        }
    ],
    "location": [
        {
            "type": "GeoProperty",
            "value": {
                "type": "Point",
                "coordinates": [13.404, 52.47, 0.0]
            },
            "observedAt": "2021-09-15T13:41:37.888Z",
            "instanceId": "urn:ngsi-ld:attribute:instance:a6d20ce2-162a-11ec-93ed-0242ac120106",
            "providedBy": {
                "object": "urn:ngsi-ld:Device:cow002",
                "type": "Relationship",
                "instanceId": "urn:ngsi-ld:attribute:instance:a6d21782-162a-11ec-93ed-0242ac120106"
            }
        },
        {
            "type": "GeoProperty",
            "value": {
                "type": "Point",
                "coordinates": [13.404, 52.471, 0.0]
            },
            "observedAt": "2021-09-15T13:41:34.568Z",
            "instanceId": "urn:ngsi-ld:attribute:instance:a4b64e00-162a-11ec-9bda-0242ac120106",
            "providedBy": {
                "object": "urn:ngsi-ld:Device:cow002",
                "type": "Relationship",
                "instanceId": "urn:ngsi-ld:attribute:instance:a4b650c6-162a-11ec-9bda-0242ac120106"
            }
        },
        {
            "type": "GeoProperty",
            "value": {
                "type": "Point",
                "coordinates": [13.404, 52.471, 0.0]
            },
            "observedAt": "2021-09-15T13:41:33.508Z",
            "instanceId": "urn:ngsi-ld:attribute:instance:a43b6d20-162a-11ec-9dd5-0242ac120106",
            "providedBy": {
                "object": "urn:ngsi-ld:Device:cow002",
                "type": "Relationship",
                "instanceId": "urn:ngsi-ld:attribute:instance:a43b70a4-162a-11ec-9dd5-0242ac120106"
            }
        }
    ],
    "species": {
        "type": "Property",
        "value": "dairy cattle",
        "instanceId": "urn:ngsi-ld:attribute:instance:c6ef9a04-1629-11ec-aced-0242ac120106"
    },
    "reproductiveCondition": {
        "type": "Property",
        "value": "active",
        "instanceId": "urn:ngsi-ld:attribute:instance:c6efa292-1629-11ec-aced-0242ac120106"
    }
}
```

<a name="list-the-last-n-sampled-values-of-an-attribute-of-an-entity"></a>

### エンティティの属性の最後の N 個のサンプル値を一覧表示

`/entities` エンドポイントからの通常のクエリ・パラメータもすべて `/temporal/entities` でサポートされています。
単一の属性の結果を取得するには、`attrs` パラメータを追加し、受け取る属性のコンマ区切りリストを含めるだけです。

この例は、エンティティ `urn:ngsi-ld:Animal:cow002` から `heartRate` の最後の3つの変更のみを示しています。

#### :two: リクエスト:

```console
curl -G -X GET 'http://localhost:8080/temporal/entities/urn:ngsi-ld:Animal:cow001' \
  -H 'NGSILD-Tenant: openiot' \
  -H 'Link: <http://context/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
  -d 'lastN=3' \
  -d 'attrs=heartRate'
```

#### レスポンス:

レスポンスは、`heartRate` の値を保持する単一の属性配列を持つ単一のエンティティです。

```json
{
    "id": "urn:ngsi-ld:Animal:cow001",
    "type": "Animal",
    "heartRate": [
        {
            "type": "Property",
            "value": 52.0,
            "observedAt": "2021-09-16T10:26:41.108Z",
            "instanceId": "urn:ngsi-ld:attribute:instance:95b296d0-16d8-11ec-90f3-0242ac120106",
            "unitCode": "5K",
            "providedBy": {
                "object": "urn:ngsi-ld:Device:cow001",
                "type": "Relationship",
                "instanceId": "urn:ngsi-ld:attribute:instance:95b2991e-16d8-11ec-90f3-0242ac120106"
            }
        },
        {
            "type": "Property",
            "value": 53.0,
            "observedAt": "2021-09-16T10:26:31.178Z",
            "instanceId": "urn:ngsi-ld:attribute:instance:8f4e048c-16d8-11ec-8e17-0242ac120106",
            "unitCode": "5K",
            "providedBy": {
                "object": "urn:ngsi-ld:Device:cow001",
                "type": "Relationship",
                "instanceId": "urn:ngsi-ld:attribute:instance:8f4e061c-16d8-11ec-8e17-0242ac120106"
            }
        },
        {
            "type": "Property",
            "value": 52.0,
            "observedAt": "2021-09-16T10:26:26.308Z",
            "instanceId": "urn:ngsi-ld:attribute:instance:8ca9e408-16d8-11ec-b8cd-0242ac120106",
            "unitCode": "5K",
            "providedBy": {
                "object": "urn:ngsi-ld:Device:cow001",
                "type": "Relationship",
                "instanceId": "urn:ngsi-ld:attribute:instance:8ca9e570-16d8-11ec-b8cd-0242ac120106"
            }
        }
    ]
}
```

<a name="simplified-temporal-representation-of-an-entity"></a>

### エンティティの簡略化されたテンポラル表現

`options=keyValues` パラメータがエンティティを単純なキーと値のペアに減らすのとほぼ同じ方法で、同等の
`options=temporalValues` は各属性を一連のタプルに減らします。エントリごとに1つの値と1つのタイムスタンプです。

簡略化された時間表現は、次のように `options` パラメータを追加することでリクエストできます:

#### :three: リクエスト:

```console
curl -G -X GET 'http://localhost:8080/temporal/entities/urn:ngsi-ld:Animal:cow001' \
  -H 'NGSILD-Tenant: openiot' \
  -H 'Link: <http://context/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
  -d 'lastN=3' \
  -d 'options=temporalValues'
```

#### レスポンス:

レスポンスは、`observedAt`_property-of-a-property_ を持つ各属性のタプルの配列を持つ単一のエンティティです。この場合、
`heartRate` と `location` が返されます。ご覧の通り、各タプルの最初の要素は、属性の履歴 `value` に対応しています。
各属性の `type` も返されます。

```json
{
    "id": "urn:ngsi-ld:Animal:cow002",
    "type": "Animal",
    "location": {
        "type": "GeoProperty",
        "values": [
            [{ "type": "Point", "coordinates": [13.416, 52.485, 0.0] }, "2021-09-16T10:59:58.790Z"],
            [{ "type": "Point", "coordinates": [13.417, 52.485, 0.0] }, "2021-09-16T10:59:54.038Z"],
            [{ "type": "Point", "coordinates": [13.417, 52.484, 0.0] }, "2021-09-16T10:59:52.138Z"]
        ]
    },
    "heartRate": {
        "type": "Property",
        "values": [
            [52.0, "2021-09-16T10:59:58.790Z"],
            [51.0, "2021-09-16T10:59:54.038Z"],
            [50.0, "2021-09-16T10:59:52.138Z"]
        ]
    }
}
```

<a name="temporal-queries-without-observedat"></a>

### observedAt なしのテンポラル・クエリ

テンポラル操作は、`observedAt` _property of a property_ クエリの使用に大きく依存しますが、`timeproperty`パラメータを
使用して静的属性に対して作成し、クエリの時間インデックスを切り替えて、`modifiedAt` に対してルックアップを作成することも
できます。

#### :four: リクエスト:

次のクエリは、群れ内の雄牛に関するデータをリクエストしています。`sex` 属性は変更されないため、`timeproperty=modifiedAt`
を使用する必要があります。

```console
curl -G -X GET 'http://localhost:8080/temporal/entities/' \
  -H 'NGSILD-Tenant: openiot' \
  -H 'Link: <http://context/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
  -d 'type=Animal' \
  -d 'pageSize=2' \
  -d 'lastN=3' \
  -d 'q=sex==%22male%22' \
  -d 'timeproperty=modifiedAt' \
  -d 'options=count' \
  -d 'attrs=sex,heartRate' \
  -d 'timerel=before' \
  -d 'timeAt=<current_time>' \

```

`timerel=before` および `timeAt=<current_time>` は必須パラメータです。`<current_time>` は `2021-09-16T11:00Z` のような
UTC 形式で表された日時です。秒とミリ秒はオプションです

#### レスポンス:

レスポンスは、示されているように、2つのリクエストされた属性とともに2つのエンティティを返します。ご覧のように、`heartRate` は、
前の3つの値を返し、`sex` は1つのプロパティを返します。単一値の静的属性は、JSON-LD 構文で指定された形式であるため、1つの要素の
配列からオブジェクトに縮小されます。

```json
[
    {
        "id": "urn:ngsi-ld:Animal:cow001",
        "type": "Animal",
        "sex": {
            "type": "Property",
            "value": "male",
            "modifiedAt": "2021-09-16T10:22:39.650Z",
            "instanceId": "urn:ngsi-ld:attribute:instance:0776b1b2-16d8-11ec-9e96-0242ac120106"
        },
        "heartRate": [
            {
                "type": "Property",
                "value": 52.0,
                "observedAt": "2021-09-16T10:26:41.108Z",
                "modifiedAt": "2021-09-16T10:26:41.108Z",
                "instanceId": "urn:ngsi-ld:attribute:instance:95b296d0-16d8-11ec-90f3-0242ac120106",
                "unitCode": "5K",
                "providedBy": {
                    "object": "urn:ngsi-ld:Device:cow001",
                    "type": "Relationship",
                    "instanceId": "urn:ngsi-ld:attribute:instance:95b2991e-16d8-11ec-90f3-0242ac120106"
                }
            },
            {
                "type": "Property",
                "value": 53.0,
                "observedAt": "2021-09-16T10:26:31.178Z",
                "modifiedAt": "2021-09-16T10:26:31.108Z",
                "instanceId": "urn:ngsi-ld:attribute:instance:8f4e048c-16d8-11ec-8e17-0242ac120106",
                "unitCode": "5K",
                "providedBy": {
                    "object": "urn:ngsi-ld:Device:cow001",
                    "type": "Relationship",
                    "instanceId": "urn:ngsi-ld:attribute:instance:8f4e061c-16d8-11ec-8e17-0242ac120106"
                }
            },
            {
                "type": "Property",
                "value": 52.0,
                "observedAt": "2021-09-16T10:26:26.308Z",
                "modifiedAt": "2021-09-16T10:26:26.108Z",
                "instanceId": "urn:ngsi-ld:attribute:instance:8ca9e408-16d8-11ec-b8cd-0242ac120106",
                "unitCode": "5K",
                "providedBy": {
                    "object": "urn:ngsi-ld:Device:cow001",
                    "type": "Relationship",
                    "instanceId": "urn:ngsi-ld:attribute:instance:8ca9e570-16d8-11ec-b8cd-0242ac120106"
                }
            }
        ]
    },
    {
        "id": "urn:ngsi-ld:Animal:cow021",
        "type": "Animal",
        "sex": {
            "type": "Property",
            "value": "male",
            "modifiedAt": "2021-09-16T10:22:39.650Z",
            "instanceId": "urn:ngsi-ld:attribute:instance:07792b40-16d8-11ec-9e96-0242ac120106"
        },
        "heartRate": [
            {
                "type": "Property",
                "value": 52.0,
                "observedAt": "2021-09-16T10:26:41.108Z",
                "modifiedAt": "2021-09-16T10:26:41.108Z",
                "instanceId": "urn:ngsi-ld:attribute:instance:95b296d0-16d8-11ec-90f3-0242ac120106",
                "unitCode": "5K",
                "providedBy": {
                    "object": "urn:ngsi-ld:Device:cow021",
                    "type": "Relationship",
                    "instanceId": "urn:ngsi-ld:attribute:instance:95b2991e-16d8-11ec-90f3-0242ac120106"
                }
            },
            {
                "type": "Property",
                "value": 53.0,
                "observedAt": "2021-09-16T10:26:31.178Z",
                "modifiedAt": "2021-09-16T10:26:31.108Z",
                "instanceId": "urn:ngsi-ld:attribute:instance:8f4e048c-16d8-11ec-8e17-0242ac120106",
                "unitCode": "5K",
                "providedBy": {
                    "object": "urn:ngsi-ld:Device:cow021",
                    "type": "Relationship",
                    "instanceId": "urn:ngsi-ld:attribute:instance:8f4e061c-16d8-11ec-8e17-0242ac120106"
                }
            },
            {
                "type": "Property",
                "value": 52.0,
                "observedAt": "2021-09-16T10:26:26.308Z",
                "modifiedAt": "2021-09-16T10:26:41.308Z",
                "instanceId": "urn:ngsi-ld:attribute:instance:8ca9e408-16d8-11ec-b8cd-0242ac120106",
                "unitCode": "5K",
                "providedBy": {
                    "object": "urn:ngsi-ld:Device:cow021",
                    "type": "Relationship",
                    "instanceId": "urn:ngsi-ld:attribute:instance:8ca9e570-16d8-11ec-b8cd-0242ac120106"
                }
            }
        ]
    }
]
```

同等の簡略化された形式は、`options=temporalValues` を設定することで取得できます。

#### :five: リクエスト:

次のクエリは、群れ内の雄牛に関するデータを要求しています。`sex` 属性は変更されないため、`timeproperty=modifiedAt`
を使用する必要があります。

```console
curl -G -X GET 'http://localhost:8080/temporal/entities/' \
  -H 'NGSILD-Tenant: openiot' \
  -H 'Link: <http://context/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
  -d 'type=Animal' \
  -d 'pageSize=2' \
  -d 'lastN=3' \
  -d 'q=sex==%22male%22' \
  -d 'timeproperty=modifiedAt' \
  -d 'options=temporalValues' \
  -d 'attrs=sex,heartRate' \
  -d 'timerel=before' \
  -d 'timeAt=<current_time>' \

```

#### レスポンス:

レスポンスは、示されているように、2つの要求された属性とともに2つのエンティティを返します。ご覧のように、`heartRate` は、
前の3つの値を返し、`sex` は1つのプロパティを返します。各タプル内の2番目の要素のタイムスタンプは `modifiedAt` プロパティ
を表します。

```json
[
    {
        "id": "urn:ngsi-ld:Animal:cow001",
        "type": "Animal",
        "heartRate": {
            "type": "Property",
            "values": [
                [52.0, "2021-09-16T10:59:59.329Z"],
                [51.0, "2021-09-16T10:59:54.169Z"],
                [50.0, "2021-09-16T10:59:52.479Z"]
            ]
        },
        "sex": {
            "type": "Property",
            "values": [["male", "2021-09-16T10:22:39.650Z"]]
        }
    },
    {
        "id": "urn:ngsi-ld:Animal:cow021",
        "type": "Animal",
        "heartRate": {
            "type": "Property",
            "values": [
                [52.0, "2021-09-16T10:59:59.375Z"],
                [51.0, "2021-09-16T10:59:54.167Z"],
                [51.0, "2021-09-16T10:59:52.441Z"]
            ]
        },
        "sex": {
            "type": "Property",
            "values": [["male", "2021-09-16T10:22:39.650Z"]]
        }
    }
]
```

<a name="pagination-operations"></a>

### ページネーション操作

テンポラル操作は非常に大きなペイロードを返す可能性があるため、リクエストの範囲を縮小することができます。この
`pageSize=2` パラメータは、2つのエンティティのみが返されることを意味します。

#### :six: リクエスト:

```console
curl -G -I -X GET 'http://localhost:8080/temporal/entities/' \
  -H 'NGSILD-Tenant: openiot' \
  -H 'Link: <http://context/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
  -d 'type=Animal' \
  -d 'pageSize=2' \
  -d 'lastN=3' \
  -d 'q=sex==%22male%22' \
  -d 'timeproperty=modifiedAt' \
  -d 'options=temporalValues,count' \
  -d 'attrs=sex,heartRate' \
  -d 'timerel=before' \
  -d 'timeAt=<current_time>'
```

API は、**206 Partial Content** HTTP コードでレスポンスし、クエリに一致するより多くのデータが利用可能であることを
示していることに注意してください。`options` パラメータに `count` を追加すると、レスポンスのヘッダに追加情報が
返されます。

#### レスポンス ヘッダ:

```text
Content-Range: date-time 2021-09-16T11:00-2021-09-16T10:22:39.650/3
Page-Size: 2
Next-Page: urn:ngsi-ld:Animal:pig001
NGSILD-Results-Count: 17
Link: <http://context/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"
Date: Thu, 16 Sep 2021 14:12:32 GMT
Content-Type: application/ld+json
content-length: 540
connection: keep-alive
```

-   **Content-Range** ヘッダには、検索タイムスタンプの全体的な範囲が記載されています
-   **Page-Size** ヘッダは、配列内の数のエントリを返します。これは、リクエストされたよりも少なくてもよいです
-   **Next-Page** ヘッダは、取得される次のエンティティを示しています
-   **NGSILD-Results-Count** は、リクエストにマッチしたエンティティの合計数を示します

上記の例では、2匹の動物を取得した場合、返されるヘッダは、17匹の雄の動物が農場に住んでおり、次に返されるエンティティは
`urn:ngsi-ld:Animal:pig001` になることを示しています。

追加の `pageAnchor` パラメータを使用して同じリクエストを行うと、次の2つのエンティティが取得されます:

#### :seven: リクエスト:

```console
curl -G -X GET 'http://localhost:8080/temporal/entities/' \
  -H 'NGSILD-Tenant: openiot' \
  -H 'Link: <http://context/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
  -d 'type=Animal' \
  -d 'pageSize=2' \
  -d 'lastN=3' \
  -d 'q=sex==%22male%22' \
  -d 'timeproperty=modifiedAt' \
  -d 'options=temporalValues,count' \
  -d 'attrs=sex,heartRate' \
  -d 'timerel=before' \
  -d 'timeAt=<current_time>' \
  -d 'pageAnchor=urn:ngsi-ld:Animal:pig001' \
```

#### レスポンス ヘッダ:

`pageAnchor` を送信した場合、追加の `Previous-Page` ヘッダが前のクエリの最初のエンティティを示すレスポンスに
追加されます。

```text
Content-Range: date-time 2021-09-16T11:00-2021-09-16T10:22:39.650/3
Page-Size: 2
Next-Page: urn:ngsi-ld:Animal:pig005
Previous-Page: urn:ngsi-ld:Animal:cow001
NGSILD-Results-Count: 17
Link: <http://context/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"
Date: Thu, 16 Sep 2021 14:27:14 GMT
Content-Type: application/ld+json
content-length: 540
connection: keep-alive
```

#### レスポンス:

レスポンス・ボディ内で、2頭の雄豚が返され、API も **206 Partial Content** HTTP コードでレスポンスしました。これは、
農場でさらに雄の動物が見つかることを示しています。

```json
[
    {
        "id": "urn:ngsi-ld:Animal:pig003",
        "type": "Animal",
        "heartRate": {
            "type": "Property",
            "values": [
                [63.0, "2021-09-16T10:59:59.201Z"],
                [62.0, "2021-09-16T10:59:53.916Z"],
                [63.0, "2021-09-16T10:59:52.453Z"]
            ]
        },
        "sex": {
            "type": "Property",
            "values": [["male", "2021-09-16T10:22:39.650Z"]]
        }
    },
    {
        "id": "urn:ngsi-ld:Animal:pig001",
        "type": "Animal",
        "heartRate": {
            "type": "Property",
            "values": [
                [61.0, "2021-09-16T10:59:59.089Z"],
                [62.0, "2021-09-16T10:59:53.726Z"],
                [63.0, "2021-09-16T10:59:52.449Z"]
            ]
        },
        "sex": {
            "type": "Property",
            "values": [["male", "2021-09-16T10:22:39.650Z"]]
        }
    }
]
```

<a name="filtering-temporal-requests-using-the-q-parameter"></a>

### `q` パラメータを使用した一時的なリクエストのフィルタリング

動物の首輪などのデバイスは、IoT Anget を介してそのデータを Context Broker に送信します。対応するエンティティは
Context Broker に保存されます。

#### :eight: リクエスト:

```console
curl -L -X GET \
  'http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:Device:pig003' \
  -H 'Link: <http://context/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
  -H 'NGSILD-Tenant: openiot'
```

#### レスポンス:

エンティティは標準の **Device** データモデルに従っており、`location` および `controlledAsset` などの属性を
持っています (つまり、デバイスを装着している **Animal** エンティティ)。動物の首輪は、`heartRate` を
監視しているため、その属性もモデルに追加されています。

```json
{
    "@context": "http://context/ngsi-context.jsonld",
    "id": "urn:ngsi-ld:Device:pig003",
    "type": "Device",
    "heartRate": {
        "value": 65,
        "type": "Property",
        "unitCode": "5K",
        "observedAt": "2021-09-16T15:24:15.781Z"
    },
    "status": {
        "value": "3",
        "type": "Property",
        "observedAt": "2021-09-16T15:24:15.781Z"
    },
    "location": {
        "value": {
            "type": "Point",
            "coordinates": [13.356, 52.511]
        },
        "type": "GeoProperty",
        "observedAt": "2021-09-16T15:24:15.781Z"
    },
    "controlledAsset": {
        "object": "urn:ngsi-ld:Animal:pig003",
        "type": "Relationship",
        "observedAt": "2021-09-16T15:24:15.781Z"
    },
    "description": {
        "value": "Animal Collar",
        "type": "Property",
        "observedAt": "2021-09-16T15:24:15.781Z"
    },
    "category": {
        "value": "sensor",
        "type": "Property",
        "observedAt": "2021-09-16T15:24:15.781Z"
    },
    "controlledProperty": {
        "value": ["heartRate", "location", "status"],
        "type": "Property",
        "observedAt": "2021-09-16T15:24:15.781Z"
    },
    "supportedProtocol": {
        "value": "ul20",
        "type": "Property",
        "observedAt": "2021-09-16T15:24:15.781Z"
    },
    "d": {
        "value": "FORAGING",
        "type": "Property",
        "observedAt": "2021-09-16T15:24:15.781Z"
    }
}
```

ご覧のとおり、返されるすべての属性には `observedAt` _property of a property_ があります。これは、任意の属性を
テンポラル・リクエストのフィルターの一部として使用できることを意味します。

次のリクエストは、動物の状態が `FORAGING` (採餌) として記述されているときに登録された `heartRate` を返し、それを身に
付けている関連する Animal エンティティも返します。

#### :nine: リクエスト:

```console
curl -G -X GET 'http://localhost:8080/temporal/entities/' \
  -H 'NGSILD-Tenant: openiot' \
  -H 'Link: <http://context/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
  -d 'type=Device' \
  -d 'q=d==%22FORAGING%22' \
  -d 'attrs=heartRate,controlledAsset' \
  -d 'options=temporalValues' \
  -d 'timerel=before' \
  -d 'timeAt=<current_time>' \
  -d 'pageSize=2' \
  -d 'lastN=2'
```

#### レスポンス:

レスポンスは、リクエストされた属性を簡略化されたテンポラル形式で返します。

```json
[
    {
        "id": "urn:ngsi-ld:Device:pig001",
        "type": "Device",
        "heartRate": {
            "type": "Property",
            "values": [
                [64.0, "2021-09-16T15:45:58.455Z"],
                [63.0, "2021-09-16T15:45:53.586Z"]
            ]
        },
        "controlledAsset": {
            "type": "Relationship",
            "objects": [
                ["urn:ngsi-ld:Animal:pig001", "2021-09-16T15:45:58.455Z"],
                ["urn:ngsi-ld:Animal:pig001", "2021-09-16T15:45:53.586Z"]
            ]
        }
    },
    {
        "id": "urn:ngsi-ld:Device:pig002",
        "type": "Device",
        "heartRate": {
            "type": "Property",
            "values": [
                [64.0, "2021-09-16T15:50:00.659Z"],
                [65.0, "2021-09-16T15:49:56.099Z"]
            ]
        },
        "controlledAsset": {
            "type": "Relationship",
            "objects": [
                ["urn:ngsi-ld:Animal:pig002", "2021-09-16T15:50:00.659Z"],
                ["urn:ngsi-ld:Animal:pig002", "2021-09-16T15:49:56.099Z"]
            ]
        }
    }
]
```

<a name="geofencing-temporal-requests-using-the-georel-parameter"></a>

### `georel` パラメータを使用したテンポラル・リクエストのジオフェンス

`location` を持つエンティティを `georel` パラメータを使用してジオフィルタリングできるのと同じ方法で、時間の経過
とともに観察される GeoProperty 属性に対してテンポラル・クエリを実行できます。以前の **Device** クエリからわかるように、
各動物の首輪の `location` 属性には `observedAt` _property-of-a-property_ があるため、時間の経過とともに場所を
追跡するために使用できます。

次のリクエストは、動物の状態が定点から800メートル未満のときに登録された `heartRate` を返し、それを身に付けている
関連する動物エンティティも返します。

#### :nine: リクエスト:

```console
curl -G -X GET 'http://localhost:8080/temporal/entities/' \
  -H 'NGSILD-Tenant: openiot' \
  -H 'Link: <http://context/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
  -d 'type=Device' \
  -d 'georel=near%3BmaxDistance==800' \
  -d 'geometry=Point' \
  -d 'coordinates=%5B13.364,52.52%5D' \
  -d 'attrs=heartRate,controlledAsset' \
  -d 'options=temporalValues' \
  -d 'timerel=before' \
  -d 'timeAt=<current_time>' \
  -d 'pageSize=2' \
  -d 'lastN=3'
```

#### レスポンス:

レスポンスは、リクエストされた属性を簡略化されたテンポラル形式で返します。

```json
[
    {
        "id": "urn:ngsi-ld:Device:pig001",
        "type": "Device",
        "heartRate": {
            "type": "Property",
            "values": [
                [66.0, "2021-09-16T15:21:59.574Z"],
                [66.0, "2021-09-16T15:21:54.788Z"],
                [64.0, "2021-09-16T15:21:44.759Z"]
            ]
        },
        "controlledAsset": {
            "type": "Relationship",
            "objects": [
                ["urn:ngsi-ld:Animal:pig001", "2021-09-16T15:21:59.574Z"],
                ["urn:ngsi-ld:Animal:pig001", "2021-09-16T15:21:54.788Z"],
                ["urn:ngsi-ld:Animal:pig001", "2021-09-16T15:21:44.759Z"]
            ]
        }
    },
    {
        "id": "urn:ngsi-ld:Device:pig002",
        "type": "Device",
        "heartRate": {
            "type": "Property",
            "values": [
                [64.0, "2021-09-16T16:23:57.731Z"],
                [63.0, "2021-09-16T16:23:52.839Z"],
                [62.0, "2021-09-16T16:23:47.102Z"]
            ]
        },
        "controlledAsset": {
            "type": "Relationship",
            "objects": [
                ["urn:ngsi-ld:Animal:pig002", "2021-09-16T16:23:57.731Z"],
                ["urn:ngsi-ld:Animal:pig002", "2021-09-16T16:23:52.839Z"],
                ["urn:ngsi-ld:Animal:pig002", "2021-09-16T16:23:47.102Z"]
            ]
        }
    }
]
```

<a name="next-steps"></a>

# 次のステップ

高度な機能を追加することで、アプリケーションに複雑さを加える方法を知りたいですか？ このシリーズの
[他のチュートリアル](https://www.letsfiware.jp/ngsi-ld-tutorials)を読むことで見つけることができます

---

## License

[MIT](LICENSE) © 2021-2023 FIWARE Foundation e.V.
