[![FIWARE Banner](https://fiware.github.io/tutorials.Short-Term-History/img/fiware.png)](https://www.fiware.org/developers)

このチュートリアルでは、Mongo-DB データベースから傾向データを取得するために使用される Generic Enabler である [FIWARE STH-Comet](https://fiware-sth-comet.readthedocs.io/) について紹介します。チュートリアルでは、[前のチュートリアル](https://github.com/Fiware/tutorials.IoT-Agent)で接続した IoT センサをアクティブにし、それらのセンサからの測定値をデータベースに保存し、そのデータの時間ベースの集計を取得します。

このチュートリアルでは、全体で [cUrl](https://ec.haxx.se/) コマンドを使用していますが、[Postman documentation](http://fiware.github.io/tutorials.Short-Term-History/) も利用できます。

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/4824d3171f823935dcab)


# 内容

- [時系列データのクエリ (Mongo-DB)](#querying-time-series-data-mongo-db)
  * [時系列データの解析](#analyzing-time-series-data)
- [アーキテクチャ](#architecture)
- [前提条件](#prerequisites)
  * [Docker と Docker Compose](#docker-and-docker-compose)
  * [Cygwin for Windows](#cygwin-for-windows)
- [起動](#start-up)
- [*最小*モード (STH-Comet のみ)](#minimal-mode-sth-comet-only)
  * [データベース・サーバの設定](#database-server-configuration)
  * [STH-Comet の設定](#sth-comet-configuration)
  * [*最小*モード - 起動](#minimal-mode---start-up)
    + [STH-Comet - サービスの健全性のチェック](#sth-comet---checking-service-health)
    + [コンテキスト・データの生成](#generating-context-data)
  * [*最小*モード - コンテキスト変更に対する STH-Comet のサブスクリプション](#minimal-mode---subscribing-sth-comet-to-context-changes)
    + [STH-Comet - モーション・センサのカウント・イベントの集計](#sth-comet---aggregate-motion-sensor-count-events)
    + [STH-Comet - ランプの明度のサンプリング](#sth-comet---sample-lamp-luminosity)
- [時系列データのクエリ](#time-series-data-queries)
  * [前提条件](#prerequisites-1)
    + [サブスクリプションが存在することを確認](#check-that-subscriptions-exist)
  * [オフセット，制限，ページネーション](#offsets-limits-and-pagination)
    + [最初の N 個のサンプリング値をリスト](#list-the-first-n-sampled-values)
    + [オフセットで N 個のサンプリング値をリスト](#list-n-sampled-values-at-an-offset)
    + [最新の N 個のサンプリング値をリスト](#list-the-latest-n-sampled-values)
  * [期間クエリ](#time-period-queries)
    + [一定期間にわたる値の合計をリスト](#list-the-sum-of-values-over-a-time-period)
    + [一定期間にわたる値の最小値をリスト](#list-the-minimum-of-a-value-over-a-time-period)
    + [一定期間にわたる値の最大値をリスト](#list-the-maximum-of-a-value-over-a-time-period)
- [*正規*モード (Cygnus + STH-Comet)](#formal-mode-cygnus--sth-comet)
  * [データベース・サーバの設定](#database-server-configuration-1)
  * [STH-Comet の設定](#sth-comet-configuration-1)
  * [Cygnus の設定](#cygnus-configuration)
  * [*正規*モード - 起動](#formal-mode---start-up)
    + [STH-Comet - サービスの健全性のチェック](#sth-comet---checking-service-health-1)
    + [Cygnus - サービスの健全性のチェック](#cygnus---checking-service-health)
    + [コンテキスト・データの生成](#generating-context-data-1)
  * [*正規*モード - コンテキスト変更に対する Cygnus のサブスクリプション](#formal-mode---subscribing-cygnus-to-context-changes)
    + [Cygnus - モーション・センサのカウント・イベントの集計](#cygnus---aggregate-motion-sensor-count-events)
    + [Cygnus - ランプの明度のサンプリング](#cygnus---sample-lamp-luminosity)
  * [*正規*モード - 時系列データのクエリ](#formal-mode---time-series-data-queries)
- [プログラミングによる時系列データへのアクセス](#accessing-time-series-data-programmatically)
- [次のステップ](#next-steps)


<a name="querying-time-series-data-mongo-db"></a>
# 時系列データのクエリ (Mongo-DB)

> "The *"moment"* has no yesterday or tomorrow. It is not the result of thought and therefore has no time."
>
> — Bruce Lee


FIWARE プラットフォーム内では、**Orion Context Broker** と **Cygnus** Generic Enabler の組み合わせを使用して、履歴コンテキスト・データを Mongo-DB のようなデータベースに保存できます。これにより、選択したデータベースに一連のデータ・ポイントが書き込まれます。各タイムスタンプ付きデータ・ポイントは、所与の瞬間におけるコンテキスト・エンティティの状態を表します。個々のデータ点はそれ自体では無意味であり、最大値、最小値および傾向などの意味のある統計値が観測されることができるのは、一連のデータ点を組み合わせたときだけです。

傾向データの作成と分析は、コンテキスト駆動型システムの一般的な要件です。したがって、FIWARE プラットフォームは、Mong-DB に永続化された時系列データの永続化と解釈の問題に特化した Generic Enabler ([STH-Comet](https://fiware-sth-comet.readthedocs.io/)) を提供します。**STH-Comet** 自体は2つのモードで使用できます : 

* *最小*モードでは、**STH-Comet**は、リクエストされたときに、データの収集とデータの解釈の両方に責任があります
* *正規*モードでは、データの収集が **Cygnus** に委任され、**STH-Comet** は、単に既存のデータベースから読み込みます

2つの動作モードのうち、*正規*モードはより柔軟ですが、*最小*モードは設定が簡単です。両者の主な違いを以下の表に要約します。


|                                                         | *最小* モード                | *正規*モード              |
|---------------------------------------------------------|-------------------------------|----------------------------|
| システムは適切にセットアップしやすいですか？ | 1つの構成だけがサポートされています | 高度な設定が可能です - 複雑な設定 |
| どのコンポーネントがデータの永続性を担っていますか？ | **STH-Comet** | **Cygnus** |
| **STH-Comet** の役割は何ですか？ | データの読み書き | データ読み取り専用 |
| **Cygnus** の役割は何ですか？ | 使用しません | データ書き込み専用 |
| データはどこに集約されていますか？ | **STH-Comet** にのみ接続された Mongo-DB データベース | **Cygnus** と **STH-Comet** の両方に接続された Mongo-DB データベース |
| 他のデータベースを使用するようにシステムを構成できますか？ | いいえ | はい |
| ソリューションは簡単に拡張できますか？ | 簡単に拡張できません - 単純なシステムで使用します | スケールアップが容易です - 複雑なシステムに使用できます |
| システムは高いスループットに対処できますか？ | いいえ - スループットが低い場所で使用します | はい - スループットが高い場所で使用します |


<a name="analyzing-time-series-data"></a>
## 時系列データの解析

時系列データ分析を適切に使用するかどうかは、ユースケースと受け取るデータ測定の信頼性によって異なります。時系列データ分析を使用すると、次のような質問に答えることができます。

* 一定期間内のデバイスの最大測定値はどれくらいでしたか？
* 一定期間内のデバイスの平均測定値はどれくらいでしたか？
* 一定期間内にデバイスから送信された測定値の合計はどれくらいですか？

また、個々のデータポイントの重要性を減らして、スムージングによって外れ値を除外するために使用することもできます。


#### デバイス・モニタ

このチュートリアルの目的のために、一連のダミー IoT デバイスが作成され、Context Broker に接続されます。使用しているアーキテクチャとプロトコルの詳細は、[IoT Sensors チュートリアル](https://github.com/Fiware/tutorials.IoT-Sensors)にあります。各デバイスの状態は、次の UltraLight デバイス・モニタの Web ページで確認できます : `http://localhost:3000/device/monitor`

![FIWARE Monitor](https://fiware.github.io/tutorials.Short-Term-History/img/device-monitor.png)

#### デバイス履歴

**STH-Comet** がデータの集計を開始すると、各デバイスの履歴の状態は、デバイス履歴の Web ページに表示されます : `http://localhost:3000/device/history/urn:ngsi-ld:Store:001`

![](https://fiware.github.io/tutorials.Short-Term-History/img/history-graphs.png)


<a name="architecture"></a>
# アーキテクチャ

このアプリケーションは、[前のチュートリアル](https://github.com/Fiware/tutorials.IoT-Agent/)で作成したコンポーネント と ダミー IoT デバイスをベースにしています。[Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/)，[IoT Agent for Ultralight 2.0](http://fiware-iotagent-ul.readthedocs.io/en/latest/)，[STH-Comet](http://fiware-cygnus.readthedocs.io/en/latest/)，および [Cygnus](http://fiware-cygnus.readthedocs.io/en/latest/) の 3つまたは4つの FIWARE コンポーネントをシステムの構成に応じて使用します。

したがって、全体的なアーキテクチャは次の要素で構成されます :

* 4つの **FIWARE Generic Enablers** :
  * FIWARE [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) は、[NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2) を使用してリクエストを受信します
  * FIWARE [IoT Agent for Ultralight 2.0](http://fiware-iotagent-ul.readthedocs.io/en/latest/) は、Ultralight 2.0 形式のダミー IoT デバイスからノース・バウンドの測定値を受信し、Context Broker の[NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2) リクエストに変換してコンテキスト・エンティティの状態を変更します
  * FIWARE [STH-Comet](http://fiware-sth-comet.readthedocs.io/) は以下を行います : 
    + 時間ベースのデータ・クエリを解釈します
    + コンテキストの変更をサブスクライブして、**Mongo-DB** データベースに保存します (*最小*モードのみ)
  * FIWARE [Cygnus](http://fiware-cygnus.readthedocs.io/en/latest/) は、コンテキストの変更をサブスクライブして、**Mongo-DB** データベースに保存します (*正規*モードのみ)

> :information_source: **注:** : **Cygnus** は、**STH-Comet** が*正規*モードで設定されている場合にのみ使用されます。

* [MongoDB](https://www.mongodb.com/) データベース : 
  * **Orion Context Broker**が、データ・エンティティ、サブスクリプション、レジストレーションなどのコンテキスト・データ情報を保持するために使用します
  * デバイスの URLs や Keys などのデバイス情報を保持するために **IoT Agent** によって使用されます
  * 時間ベースの履歴コンテキスト・データを保持するデータ・シンクとして使用されます
     + *最小*モード - これは **STH-Comet** によって読み取られ、追加されます
     + *正規*モード - これは **Cygnus** によって追加され、**STH-Comet** によって読み取られます
* 3つの**コンテキストプロバイダ** :
  * **在庫管理フロントエンド**は、このチュートリアルで使用していません。これは以下を行います :
     + 店舗情報を表示し、ユーザーがダミー IoT デバイスと対話できるようにします
     + 各店舗で購入できる商品を表示します
     + ユーザが製品を購入して在庫数を減らすことを許可します
  * HTTP 上で動作する [Ultralight 2.0](http://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual) プロトコルを使用して、[ダミー IoT デバイス](https://github.com/Fiware/tutorials.IoT-Sensors)のセットとして機能する Web サーバ
  * このチュートリアルでは、**コンテキスト・プロバイダの NGSI proxy** は使用しません。これは以下を行います :
     + [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2) を使用してリクエストを受信します
     + 独自の API を独自のフォーマットで使用して、公開されているデータ・ソースへのリクエストを行います
     + [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2) 形式でコンテキスト・データ を Orion Context Broker に返します

要素間のすべての対話は HTTP リクエストによって開始されるため、エンティティはコンテナ化され、公開されたポートから実行されます。

*最小*構成と*正規*構成の特定のアーキテクチャについては、以下で説明します。

<a name="prerequisites"></a>
# 前提条件

<a name="docker-and-docker-compose"></a>
## Docker and Docker Compose 

物事を単純にするために、両方のコンポーネントが [Docker](https://www.docker.com) を使用して実行されます。**Docker** は、さまざまコンポーネントをそれぞれの環境に分離することを可能にするコンテナ・テクノロジです。

* Docker Windows にインストールするには、[こちら](https://docs.docker.com/docker-for-windows/)の手順に従ってください
* Docker Mac にインストールするには、[こちら](https://docs.docker.com/docker-for-mac/)の手順に従ってください
* Docker Linux にインストールするには、[こちら](https://docs.docker.com/install/)の手順に従ってください

**Docker Compose** は、マルチコンテナ Docker アプリケーションを定義して実行するためのツールです。[YAML file](https://raw.githubusercontent.com/Fiware/tutorials.Getting-Started/master/docker-compose.yml) ファイルは、アプリケーションのために必要なサービスを設定する使用されています。つまり、すべてのコンテナ・サービスは1つのコマンドで呼び出すことができます。Docker Compose は、デフォルトで Docker for Windows とDocker for Mac の一部としてインストールされますが、Linux ユーザは[ここ](https://docs.docker.com/compose/install/)に記載されている手順に従う必要があります。

<a name="cygwin-for-windows"></a>
## Cygwin for Windows

シンプルな bash スクリプトを使用してサービスを開始します。Windows ユーザは [cygwin](http://www.cygwin.com/) をダウンロードして、Windows 上の Linux ディストリビューションと同様のコマンドライン機能を提供する必要があります。


<a name="start-up"></a>
# 起動

開始する前に、必要な Docker イメージをローカルで取得または構築しておく必要があります。リポジトリを複製し、以下のコマンドを実行して必要なイメージを作成してください :

```console
git clone git@github.com:Fiware/tutorials.Short-Term-History.git
cd tutorials.Short-Term-History

./services create
``` 

>**注** `context-provider` イメージはまだ Docker hub にプッシュされていません。続行する前に Docker ソースをビルドできないと、次のエラーが発生します :
>
>```
>Pulling context-provider (fiware/cp-web-app:latest)...
>ERROR: The image for the service you're trying to recreate has been removed.
>```


その後、リポジトリ内で提供される [services](https://github.com/Fiware/tutorials.Historic-Context/blob/master/services) Bash スクリプトを実行することによって、コマンドラインからすべてのサービスを初期化することができます :

```console
./services <command>
``` 

ここで、`<command>` は、起動するモードによって異なります。このコマンドは、前のチュートリアルのシード・データをインポートし、起動時にダミー IoT センサをプロビジョニングします。

>:information_source: **注:** クリーンアップをやり直したい場合は、次のコマンドを使用して再起動することができます :
>
>```console
>./services stop
>``` 
>

<a name="minimal-mode-sth-comet-only"></a>
# *最小*モード (STH-Comet のみ)

*最小*構成で、**STH-Comet** は、履歴コンテキスト・データを永続化するために使用され、また、時間ベースのクエリを作成するために使用されます。すべての操作は同じポート `8666` で行われます。標準 `27017` ポートをリッスンする MongoDB インスタンスは、**Orion Context Broker** および **IoT Agent** に関連するデータを保持するだけでなく、履歴データを保持するためにも使用されます。全体的なアーキテクチャを以下に示します :

![](https://fiware.github.io/tutorials.Short-Term-History/img/sth-comet.png)

<a name="database-server-configuration"></a>
## データベース・サーバの設定

```yaml
  mongo-db:
    image: mongo:3.6
    hostname: mongo-db
    container_name: db-mongo
    ports:
        - "27017:27017"
    networks:
        - default
    command: --bind_ip_all --smallfiles
```

<a name="sth-comet-configuration"></a>
## STH-Comet の設定

```yaml
  sth-comet:
    image: fiware/sth-comet
    hostname: sth-comet
    container_name: fiware-sth-comet
    depends_on:
        - mongo-db
    networks:
        - default
    ports:
        - "8666:8666"
    environment:
        - STH_HOST=0.0.0.0
        - STH_PORT=8666
        - DB_PREFIX=sth_
        - DB_URI=mongo-db:27017
        - LOGOPS_LEVEL=DEBUG
```

`sth-comet` コンテナは、1つのポートで待機しています : 

* STH-Comet のポートの操作 - `8666` は、サービスが Orion Context Broker からの通知と、cUrl または Postman からの時間ベースのクエリのリクエストをリッスンするポートです

`sth-comet` コンテナは、次に示すように環境変数によって制御されます :

| キー       | 値              | 説明      |
|------------|-----------------|-----------|
|STH_HOST    |`0.0.0.0`        | STH-Comet がホストされているアドレス - このコンテナ内では、ローカルマシン上のすべての IPv4 アドレスを意味します |
|STH_PORT    |`8666`           | STH-Comet がリッスンするオペレーション・ポート。コンテキスト・データの変更をサブスクライブするときにも使用されます |
|DB_PREFIX   |`sth_`           | 指定されていない場合は、各データベースのエンティティに追加されたプレフィックス |
|DB_URI      |`mongo-db:27017` | STH-Comet が履歴コンテキスト・データを永続化するために接続する Mongo-DB サーバ |
|LOGOPS_LEVEL|`DEBUG`          | STH-Comet のログレベル |


<a name="minimal-mode---start-up"></a>
## *最小*モード - 起動

**STH-Comet** のみを使用して、*最小*構成でシステムを起動するには、次のコマンドを実行します :

```console
./services sth-comet
``` 

<a name="sth-comet---checking-service-health"></a>
### STH-Comet - サービスの健全性のチェック
 
STH-Comet が実行されると、公開されている `STH_PORT` ポートへの HTTP リクエストを行うことで状態を確認できます。レスポンスがブランクの場合は、通常、**STH-Comet** が実行されていないか、別のポートでリッスンしているためです。


#### :one: リクエスト :

```console
curl -X GET \
  'http://localhost:8666/version'
```

#### レスポンス :

レスポンスは次のようになります :

```json
{
    "version": "2.3.0-next"
}
```

>**トラブルシューティング :** レスポンスがブランクの場合はどうなりますか？
>
> * Docker コンテナが動作していることを確認するには :
>
>```bash
>docker ps
>```
>
>いくつかのコンテナが走っているのを確認してください。`sth-comet` または `cygnus` が実行されていない場合は、必要に応じてコンテナを再起動できます。

<a name="generating-context-data"></a>
### コンテキスト・データの生成

このチュートリアルでは、コンテキストが定期的に更新されるシステムを監視する必要があります。ダミー IoT センサを使用してこれを行うことができます。`http://localhost:3000/device/monitor` でデバイス・モニタのページを開き、**スマート・ドア**のロックを解除し、**スマート・ランプ**をオンにします。これは、ドロップ・ダウン・リストから適切なコマンドを選択し、`send` ボタンを押すことによって行うことができます。デバイスからの測定の流れは、同じページに表示されます。

![](https://fiware.github.io/tutorials.Short-Term-History/img/door-open.gif)


<a name="minimal-mode---subscribing-sth-comet-to-context-changes"></a>
## *最小*モード - コンテキスト変更に対する STH-Comet のサブスクリプション

*最小*モードで動的コンテキスト・システムが起動して実行されると、**STH-Comet** はコンテキストの変更を通知する必要があります。したがって、これらの変更を **STH-Comet** に通知するために **Orion Context Broker** にサブスクリプションを設定する必要があります。サブスクリプションの詳細は、監視対象のデバイスとサンプリング・レートによって異なります。

<a name="sth-comet---aggregate-motion-sensor-count-events"></a>
### STH-Comet - モーション・センサのカウント・イベントの集計

**モーション・センサ**の変化率は、現実世界の事象によって引き起こされます。結果を集約するためには、すべてのイベントを受け取る必要があります。

これは、**Orion Context Broker** の `/v2/subscription` エンドポイントに POST リクエストをすることで行われます。

* `fiware-service` と `fiware-servicepath` ヘッダは、サブスクリプションをフィルタリングして、接続された IoT センサからの測定値のみをリッスンためにするために使用されます
* リクエストのボディの `idPattern` は、**STH-Comet** にすべての**モーション・センサ**のデータの変更が通知されるようにします
* 通知 `url` は設定された `STH_PORT` と一致する必要があります
* **STH-Comet** は、現在、古い NGSI v1 形式の通知のみを受け付けるため、`attrsFormat=legacy` が必要です

#### :two: リクエスト :

```console
curl -iX POST \
  'http://localhost:1026/v2/subscriptions/' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
  "description": "Notify STH-Comet of all Motion Sensor count changes",
  "subject": {
    "entities": [
      {
        "idPattern": "Motion.*"
      }
    ],
    "condition": {"attrs": ["count"] }
  },
  "notification": {
    "http": {
      "url": "http://sth-comet:8666/notify"
    },
    "attrs": [
      "count"
    ],
    "attrsFormat": "legacy"
  }
}'
```

<a name="sth-comet---sample-lamp-luminosity"></a>
### STH-Comet - ランプの明度のサンプリング

**スマート・ランプ**の明るさは常に変化していますので、最小値や最大値、変化率などの関連する統計値を計算するために値を**サンプリング**するだけです。

これは、**Orion Context Broker** の `/v2/subscription` エンドポイントに POST リクエストを行い、リクエストのボディに `throttling` 属性 を含めることによって行われます。

* `fiware-service` と `fiware-servicepath` ヘッダは、サブスクリプションをフィルタリングして、接続された IoT センサからの測定値のみをリッスンためにするために使用されます
* リクエストのボディの `idPattern` は、**STH-Comet** にすべての**スマート・ランプ**のデータの変更が通知されるようにします
* 通知 `url` は設定された `STH_PORT` と一致する必要があります
* **STH-Comet** は、現在、古い NGSI v1 形式の通知のみを受け付けるため、`attrsFormat=legacy` が必要です
* `throttling` 値は、変更がサンプリングされる割合を定義します

> :information_source: **注 :** サブスクリプションの throttling が順次更新として期待どおりに行われないため、注意が必要です。
> たとえば、Ultra Lightデバイスが測定値 `t|20|l|1200` を送信すると、それは単一アトミックのコミットになり、両方の属性は **STH-Comet** への通知に含まれます。しかしながら、デバイスが `t|20#l|1200` を送信している場合、これは2つのアトミック・コミットとして扱われます。`t` の最初の変更に対して通知が送られますが、エンティティがサンプリング期間内に最近更新されたので、`l`の第2の変更は無視されます。

#### :three: リクエスト :

```console
curl -iX POST \
  'http://localhost:1026/v2/subscriptions/' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
  "description": "Notify Cygnus to sample Lamp luminosity every five seconds",
  "subject": {
    "entities": [
      {
        "idPattern": "Lamp.*"
      }
    ],
    "condition": {
      "attrs": [
        "luminosity"
      ]
    }
  },
  "notification": {
    "http": {
      "url": "http://sth-comet:8666/notify"
    },
    "attrs": [
      "luminosity"
    ],
    "attrsFormat": "legacy"
  },
  "throttling": 5
}'
```

<a name="time-series-data-queries"></a>
# 時系列データのクエリ

このセクションのクエリは、*最小*モードまたは*正規*モードのいずれかを使用して、**STH-Comet** をすでに接続しており、いくつかのデータを収集していることを前提としています。

<a name="prerequisites-1"></a>
## 前提条件

**STH-Comet** は、十分なデータ・ポイントが既にシステム内に集約されている場合にのみ、時系列データを取り出すことができます。**スマート・ドア**がロック解除されていて、**スマート・ランプ**がオンになっており、サブスクリプションが登録されていることを確認してください。データは、チュートリアルの前に少なくとも1分間は収集する必要があります。

<a name="check-that-subscriptions-exist"></a>
### サブスクリプションが存在することを確認

`fiware-service` および `fiware-servicepath` ヘッダはクエリに設定し、サブスクリプションを設定するときに使用する値と一致する必要があることに注意してください。

#### :four: リクエスト :

```console
curl -X GET \
  'http://localhost:1026/v2/subscriptions/' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' 
}'
```

結果は空であってはいけません。

<a name="offsets-limits-and-pagination"></a>
## オフセット，制限，ページネーション

<a name="list-the-first-n-sampled-values"></a>
### 最初の N 個のサンプリング値をリスト

この例は、`Lamp:001` から最初に3つのサンプリングされた `luminosity` 値を示しています。

コンテキスト・エンティティ属性の短期履歴を取得するには、GET リクエストを送信します :
`../STH/v1/contextEntities/type/<Entity>/id/<entity-id>/attributes/<attribute>`

`hLimit` パラメータは、N 値に結果を制限します。`hOffset=0` は、最初の値から開始します。

#### :five: リクエスト :

```console
curl -X GET \
  'http://localhost:8666/STH/v1/contextEntities/type/Lamp/id/Lamp:001/attributes/luminosity?hLimit=3&hOffset=0' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```


#### レスポンス :

```json
{
    "contextResponses": [
        {
            "contextElement": {
                "attributes": [
                    {
                        "name": "luminosity",
                        "values": [
                            {
                                "recvTime": "2018-06-21T12:20:19.841Z",
                                "attrType": "Integer",
                                "attrValue": "1972"
                            },
                            {
                                "recvTime": "2018-06-21T12:20:20.819Z",
                                "attrType": "Integer",
                                "attrValue": "1982"
                            },
                            {
                                "recvTime": "2018-06-21T12:20:29.923Z",
                                "attrType": "Integer",
                                "attrValue": "1937"
                            }
                        ]
                    }
                ],
                "id": "Lamp:001",
                "isPattern": false,
                "type": "Lamp"
            },
            "statusCode": {
                "code": "200",
                "reasonPhrase": "OK"
            }
        }
    ]
}
```

<a name="list-n-sampled-values-at-an-offset"></a>
### オフセットで N 個のサンプリング値をリスト

この例では、`Motion:001` からの4番目、5番目、6番目のサンプリングされた `count` 値を示しています。

コンテキスト・エンティティ属性の短期履歴を取得するには、GET リクエストを送信します :
`../STH/v1/contextEntities/type/<Entity>/id/<entity-id>/attributes/<attribute>`

`hLimit` パラメータは、N 値に結果を制限します。`hOffset` は、ゼロ以外の値に設定すると N 番目の測定から開始します。

#### :six: リクエスト :

```console
curl -X GET \
  'http://localhost:8666/STH/v1/contextEntities/type/Motion/id/Motion:001/attributes/count?hLimit=3&hOffset=3' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```


#### レスポンス :

```json
{
    "contextResponses": [
        {
            "contextElement": {
                "attributes": [
                    {
                        "name": "count",
                        "values": [
                            {
                                "recvTime": "2018-06-21T12:37:00.358Z",
                                "attrType": "Integer",
                                "attrValue": "1"
                            },
                            {
                                "recvTime": "2018-06-21T12:37:01.368Z",
                                "attrType": "Integer",
                                "attrValue": "0"
                            },
                            {
                                "recvTime": "2018-06-21T12:37:07.461Z",
                                "attrType": "Integer",
                                "attrValue": "1"
                            }
                        ]
                    }
                ],
                "id": "Motion:001",
                "isPattern": false,
                "type": "Motion"
            },
            "statusCode": {
                "code": "200",
                "reasonPhrase": "OK"
            }
        }
    ]
}
```

<a name="list-the-latest-n-sampled-values"></a>
### 最新の N 個のサンプリング値をリスト

この例は、`Motion:001` からの最新の3つのサンプリングされた `count` 値を示しています。

コンテキスト・エンティティ属性の短期履歴を取得するには、GET リクエストを送信します : 
`../STH/v1/contextEntities/type/<Entity>/id/<entity-id>/attributes/<attribute>`

`lastN` パラメータが設定された場合は、その結果は、N 個の最新の測定結果のみを返します。

#### :seven: リクエスト :

```console
curl -X GET \
  'http://localhost:8666/STH/v1/contextEntities/type/Motion/id/Motion:001/attributes/count?lastN=3' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```


#### レスポンス :

```json
{
    "contextResponses": [
        {
            "contextElement": {
                "attributes": [
                    {
                        "name": "count",
                        "values": [
                            {
                                "recvTime": "2018-06-21T12:47:28.377Z",
                                "attrType": "Integer",
                                "attrValue": "0"
                            },
                            {
                                "recvTime": "2018-06-21T12:48:08.930Z",
                                "attrType": "Integer",
                                "attrValue": "1"
                            },
                            {
                                "recvTime": "2018-06-21T12:48:13.989Z",
                                "attrType": "Integer",
                                "attrValue": "0"
                            }
                        ]
                    }
                ],
                "id": "Motion:001",
                "isPattern": false,
                "type": "Motion"
            },
            "statusCode": {
                "code": "200",
                "reasonPhrase": "OK"
            }
        }
    ]
}
```

<a name="time-period-queries"></a>
##  期間クエリ

<a name="list-the-sum-of-values-over-a-time-period"></a>
### 一定期間にわたる値の合計をリスト

この例では、1分ごとに`Motion:001`から 合計 `count` 値を示しています。

コンテキスト・エンティティ属性の短期履歴を取得するに、GET リクエストを送信します :
`../STH/v1/contextEntities/type/<Entity>/id/<entity-id>/attributes/<attribute>`

`aggrMethod` パラメータは、時系列上に実行する集計のタイプを決定します。`aggrPeriod` は、`second`, `minute`, `hour` または `day` のいずれかです。

常にデータ収集の頻度に基づいて最も適切な期間を選択してください。 `Motion:001` は、1分ごとに数回変化しているので、`minute` が選択されます。

#### :eight: リクエスト :

```console
curl -X GET \
  'http://localhost:8666/STH/v1/contextEntities/type/Motion/id/Motion:001/attributes/count?aggrMethod=sum&aggrPeriod=minute' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```


#### レスポンス :

```json
{
    "contextResponses": [
        {
            "contextElement": {
                "attributes": [
                    {
                        "name": "count",
                        "values": [
                            {
                                "_id": {
                                    "attrName": "count",
                                    "origin": "2018-06-21T12:00:00.000Z",
                                    "resolution": "minute"
                                },
                                "points": [
                                    {
                                        "offset": 37,
                                        "samples": 3,
                                        "sum": 1
                                    },
                                    {
                                        "offset": 38,
                                        "samples": 12,
                                        "sum": 6
                                    },
                                    {
                                        "offset": 39,
                                        "samples": 7,
                                        "sum": 4
                                    }, ...etc
                                ]
                            }
                        ]
                    }
                ],
                "id": "Motion:001",
                "isPattern": false,
                "type": "Motion"
            },
            "statusCode": {
                "code": "200",
                "reasonPhrase": "OK"
            }
        }
    ]
}
```

期間内の平均値のクエリは直接サポートされていません。

この例では、1分ごとの `Lamp:001` からの `luminosity` 値の合計を示します。期間内のサンプル数と組み合わせると、データから平均を計算することができます。

コンテキスト・エンティティ属性の短期履歴を取得するには、GET リクエストを送信します : 
`../STH/v1/contextEntities/type/<Entity>/id/<entity-id>/attributes/<attribute>`

`aggrMethod` パラメータは、時系列上に実行する集計のタイプを決定します。`aggrPeriod` は、`second`, `minute`, `hour` または `day` のいずれかです。


#### :nine: リクエスト :

```console
curl -X GET \
  'http://localhost:8666/STH/v1/contextEntities/type/Lamp/id/Lamp:001/attributes/luminosity?aggrMethod=sum&aggrPeriod=minute' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```

#### レスポンス :

```json
{
    "contextResponses": [
        {
            "contextElement": {
                "attributes": [
                    {
                        "name": "luminosity",
                        "values": [
                            {
                                "_id": {
                                    "attrName": "luminosity",
                                    "origin": "2018-06-21T12:00:00.000Z",
                                    "resolution": "minute"
                                },
                                "points": [
                                    {
                                        "offset": 20,
                                        "samples": 9,
                                        "sum": 17382
                                    },
                                    {
                                        "offset": 21,
                                        "samples": 8,
                                        "sum": 15655
                                    },
                                    {
                                        "offset": 22,
                                        "samples": 5,
                                        "sum": 9630
                                    },  ...etc
                                ]
                            }
                        ]
                    }
                ],
                "id": "Lamp:001",
                "isPattern": false,
                "type": "Lamp"
            },
            "statusCode": {
                "code": "200",
                "reasonPhrase": "OK"
            }
        }
    ]
}
```


<a name="list-the-minimum-of-a-value-over-a-time-period"></a>
### 一定期間にわたる値の最小値をリスト

この例では、1分ごとの `Lamp:001` からの 最小 `luminosity` 値を示しています。

コンテキスト・エンティティ属性の短期履歴を取得するには、GET リクエストを送信します :
`../STH/v1/contextEntities/type/<Entity>/id/<entity-id>/attributes/<attribute>`

`aggrMethod` パラメータは、時系列上に実行する集計のタイプを決定します。`aggrPeriod` は、`second`, `minute`, `hour` または `day` のいずれかです。

**スマート・ランプ**の光度は絶えず変化しているため、最小値を追跡することは理にかなっています。**モーション・センサ**は、バイナリ値のみを提供するため、これには適していません。

#### :one::zero: リクエスト :

```console
curl -X GET \
  'http://localhost:8666/STH/v1/contextEntities/type/Lamp/id/Lamp:001/attributes/luminosity?aggrMethod=min&aggrPeriod=minute' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```


#### レスポンス :

```json
{
    "contextResponses": [
        {
            "contextElement": {
                "attributes": [
                    {
                        "name": "luminosity",
                        "values": [
                            {
                                "_id": {
                                    "attrName": "luminosity",
                                    "origin": "2018-06-21T12:00:00.000Z",
                                    "resolution": "minute"
                                },
                                "points": [
                                    {
                                        "offset": 20,
                                        "samples": 9,
                                        "min": 1793
                                    },
                                    {
                                        "offset": 21,
                                        "samples": 8,
                                        "min": 1819
                                    },
                                    {
                                        "offset": 22,
                                        "samples": 5,
                                        "min": 1855
                                    }, ..etc
                                ]
                            }
                        ]
                    }
                ],
                "id": "Lamp:001",
                "isPattern": false,
                "type": "Lamp"
            },
            "statusCode": {
                "code": "200",
                "reasonPhrase": "OK"
            }
        }
    ]
}
```

<a name="list-the-maximum-of-a-value-over-a-time-period"></a>
### 一定期間にわたる値の最大値をリスト

この例では、1分ごとの `Lamp:001` からの最大 `luminosity` 値を示しています。

コンテキスト・エンティティ属性の短期履歴を取得するには、GET リクエストを送信します :
`../STH/v1/contextEntities/type/<Entity>/id/<entity-id>/attributes/<attribute>`

`aggrMethod` パラメータは、時系列上に実行する集計のタイプを決定します。`aggrPeriod` は、`second`, `minute`, `hour` または `day` のいずれかです。


#### :one::one: リクエスト :

```console
curl -X GET \
  'http://localhost:8666/STH/v1/contextEntities/type/Lamp/id/Lamp:001/attributes/luminosity?aggrMethod=max&aggrPeriod=minute' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```


#### レスポンス :

```json
{
    "contextResponses": [
        {
            "contextElement": {
                "attributes": [
                    {
                        "name": "luminosity",
                        "values": [
                            {
                                "_id": {
                                    "attrName": "luminosity",
                                    "origin": "2018-06-21T12:00:00.000Z",
                                    "resolution": "minute"
                                },
                                "points": [
                                    {
                                        "offset": 20,
                                        "samples": 9,
                                        "max": 2005
                                    },
                                    {
                                        "offset": 21,
                                        "samples": 8,
                                        "max": 2006
                                    },
                                    {
                                        "offset": 22,
                                        "samples": 5,
                                        "max": 1988
                                    }, ...etc
                                ]
                            }
                        ]
                    }
                ],
                "id": "Lamp:001",
                "isPattern": false,
                "type": "Lamp"
            },
            "statusCode": {
                "code": "200",
                "reasonPhrase": "OK"
            }
        }
    ]
}
```







<a name="formal-mode-cygnus--sth-comet"></a>
# *正規*モード (Cygnus + STH-Comet)

*正規*構成は **Cygnus** を使用して、[前のチュートリアル](https://github.com/Fiware/tutorials.Historic-Context)で示したのと同じ方法で、履歴コンテキスト・データを MongoDB データベースに保存します。既存の MongoDB インスタンス(標準 `27017` ポートでリスニング) は、**Orion Context Broker**，**IoT Agent**、および **Cygnus** によって永続化された履歴コンテキスト・データに関連するデータを保持するために使用されます。**STH-Comet** も同じデータベースに接続されており、そこからデータを読み込みます。全体的なアーキテクチャを以下に示します :

![](https://fiware.github.io/tutorials.Short-Term-History/img/cygnus-sth-comet.png)

<a name="database-server-configuration-1"></a>
## データベース・サーバの設定

```yaml
  mongo-db:
    image: mongo:3.6
    hostname: mongo-db
    container_name: db-mongo
    ports:
        - "27017:27017"
    networks:
        - default
    command: --bind_ip_all --smallfiles
```

<a name="sth-comet-configuration-1"></a>
## STH-Comet の設定

```yaml
  sth-comet:
    image: fiware/sth-comet
    hostname: sth-comet
    container_name: fiware-sth-comet
    depends_on:
        - mongo-db
    networks:
        - default
    ports:
        - "8666:8666"
    environment:
        - STH_HOST=0.0.0.0
        - STH_PORT=8666
        - DB_PREFIX=sth_
        - DB_URI=mongo-db:27017
        - LOGOPS_LEVEL=DEBUG
```

<a name="cygnus-configuration"></a>
## Cygnus の設定

```yaml
  cygnus:
    image: fiware/cygnus-ngsi:latest
    hostname: cygnus
    container_name: fiware-cygnus
    depends_on:
        - mongo-db
    networks:
        - default
    expose:
        - "5080"
    ports:
        - "5050:5050"
        - "5080:5080"
    environment:
        - "CYGNUS_MONGO_HOSTS=mongo-db:27017"
        - "CYGNUS_LOG_LEVEL=DEBUG"
        - "CYGNUS_SERVICE_PORT=5050"
        - "CYGNUS_API_PORT=5080"
```

`sth-comet` コンテナは、1つのポートで待機しています : 

* STH-Comet のポートの操作 - `8666` は、サービスが cUrl または Postman からの時間ベースのクエリ・リクエストをリッスンするポートです

`sth-comet` コンテナは、次に示すように、環境変数によって制御されます :

| キー       | 値              | 説明      |
|------------|-----------------|-----------|
|STH_HOST    |`0.0.0.0`        | STH-Comet がホストされているアドレス - このコンテナ内では、ローカルマシン上のすべての IPv4 アドレスを意味します |
|STH_PORT    |`8666`           | STH-Comet がリッスンするオペレーション・ポート |
|DB_PREFIX   |`sth_`           | 指定されていない場合は、各データベースのエンティティに追加されたプレフィックス |
|DB_URI      |`mongo-db:27017` | STH-Comet が履歴コンテキスト・データを永続化するために接続する Mongo-DB サーバ |
|LOGOPS_LEVEL|`DEBUG`          | STH-Comet のログレベル |

`cygnus` コンテナは、2つのポートでリッスンしています : 

* `Cygnus` のサブスクリプション・ポート - `5050` は、サービスが **Orion Context Broker** からの通知をリッスンするポートです
* `Cygnus` の管理ポート - `5080` はチュートリアルのアクセスのためだけに公開されているため、cUrl または Postman は同じネットワークの一部ではなくプロビジョニング・コマンドを実行できます

`cygnus` コンテナは、次に示すように、環境変数によって制御されます :

| キー                          | 値           | 説明      |
|-------------------------------|--------------|-----------|
|CYGNUS_MONGO_HOSTS         |`mongo-db:27017`  | Cygnus が履歴コンテキスト・データを永続化するために接続する Mongo-DB サーバのカンマ区切りリスト |
|CYGNUS_LOG_LEVEL               |`DEBUG`       | Cygnus のログレベル |
|CYGNUS_SERVICE_PORT            |`5050`        | コンテキスト・データが変更されたときに Cygnus がリッスンする通知ポート |
|CYGNUS_API_PORT                |`5080`        | Cygnus が操作上の理由でリッスンするポート |


<a name="formal-mode---start-up"></a>
## *正規*モード - 起動

**Cygnus** と **STH-Comet** を使用して*正規*構成を使用してシステムを起動するには、次のコマンドを実行します :

```console
./services cygnus
``` 

<a name="sth-comet---checking-service-health-1"></a>
### STH-Comet - サービスの健全性のチェック
 
**STH-Comet** が実行されると、公開された `STH_PORT` ポートへの HTTP  リクエストを行うことで、状態を確認できます。レスポンスがブランクの場合は、通常、**STH-Comet** が実行されていないか、別のポートでリッスンしているためです。


#### :one::two: リクエスト :

```console
curl -X GET \
  'http://localhost:8666/version'
```

#### レスポンス :

レスポンスは次のようになります :

```json
{
    "version": "2.3.0-next"
}
```

<a name="cygnus---checking-service-health"></a>
### Cygnus - サービスの健全性のチェック
 
**Cygnus** が実行されると、公開された `CYGNUS_API_PORT` ポートへの HTTP リクエストを行うことで、状況を確認できます。レスポンスがブランクの場合、これは通常、**Cygnus** が実行されていないか、別のポートでリッスンしているためです。

#### :one::three: リクエスト :

```console
curl -X GET \
  'http://localhost:5080/v1/version'
```

#### レスポンス :

The response will look similar to the following:

```json
{
    "success": "true",
    "version": "1.8.0_SNAPSHOT.ed50706880829e97fd4cf926df434f1ef4fac147"
}
```

>**トラブルシューティング :** いずれかのレスポンスがブランクの場合はどうなりますか？
>
> * Docker コンテナが動作していることを確認するには :
>
>```bash
>docker ps
>```
>
>いくつかのコンテナが走っているのを確認してください。`sth-comet` または `cygnus` が実行されていない場合は、必要に応じてコンテナを再起動できます。

<a name="generating-context-data-1"></a>
### コンテキスト・データの生成

このチュートリアルでは、コンテキストが定期的に更新されるシステムを監視する必要があります。ダミー IoT センサを使用してこれを行うことができます。`http://localhost:3000/device/monitor` でデバイス・モニタのページを開き、**スマート・ドア**のロックを解除し、**スマート・ランプ**をオンにします。これは、ドロップ・ダウン・リストから適切なコマンドを選択し、`send` ボタンを押すことによって行うことができます。デバイスからの測定の流れは、同じページに表示されます :

![](https://fiware.github.io/tutorials.Short-Term-History/img/door-open.gif)

<a name="formal-mode---subscribing-cygnus-to-context-changes"></a>
## *正規*モード - コンテキスト変更に対する Cygnus のサブスクリプション

*正規*モードでは、**Cygnus** は、履歴コンテキスト・データの永続性のために責任があります。動的コンテキスト・システムが起動したら、**Orion Context Broker** にサブスクリプションを設定して、**Cygnus** にコンテキストの変更を通知する必要があります。**STH-Comet** は永続データの読み取りにのみ使用されます。

<a name="cygnus---aggregate-motion-sensor-count-events"></a>
### Cygnus - モーション・センサのカウント・イベントの集計

**モーション・センサ**の変化率は、現実世界の事象によって引き起こされます。結果を集約するためには、すべてのイベントを受け取る必要があります。

これは、**Orion Context Broker** の `/v2/subscription` エンドポイントに POST リクエストをすることによって行われます。

* `fiware-service` および `fiware-servicepath` ヘッダは、サブスクリプションをフィルタリングして、接続されている IoT センサからの測定値だけをリッスンするために使用されます
* リクエストのボディの `idPattern` は、すべての**モーション・センサ**のデータ変更を **Cygnus** に通知します
* 通知 `url` は設定された `CYGNUS_API_PORT` と一致する必要があります
* **Cygnus** は、現在、古い NGSI v1 形式の通知のみを受け付けるため、`attrsFormat=legacy` が必要です

#### :one::four: リクエスト :

```console
curl -iX POST \
  'http://localhost:1026/v2/subscriptions/' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
  "description": "Notify Cygnus of all Motion Sensor count changes",
  "subject": {
    "entities": [
      {
        "idPattern": "Motion.*"
      }
    ],
    "condition": {
      "attrs": [
        "count"
      ]
    }
  },
  "notification": {
    "http": {
      "url": "http://cygnus:5050/notify"
    },
    "attrs": [
      "count"
    ],
    "attrsFormat": "legacy"
  }
}'
```

<a name="cygnus---sample-lamp-luminosity"></a>
### Cygnus - ランプの明度のサンプリング

**スマート・ランプ**の明るさは常に変化していますので、最小値や最大値、変化率などの関連する統計値を計算するために値をサンプリングするだけです。

これは、**Orion Context Broker** の `/v2/subscription` エンドポイントに POST リクエストを行い、リクエストのボディに `throttling` 属性 を含めることによって行われます。

* `fiware-service` および `fiware-servicepath` ヘッダは、サブスクリプションをフィルタリングして、接続された IoT センサからの測定値のみをリッスンするために使用されます
* リクエストのボディの `idPattern` は、すべての**スマート・ランプ**のデータ変更のみを **Cygnus** に通知します
* 通知 `url` は設定された `CYGNUS_API_PORT` と一致する必要があります
* **Cygnus** は、現在、古い NGSI v1 形式の通知のみを受け付けるため、`attrsFormat=legacy` が必要です
* `throttling` 値は、変更がサンプリングされる割合を定義します

#### :one::five: リクエスト :

```console
curl -iX POST \
  'http://localhost:1026/v2/subscriptions/' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
  "description": "Notify Cygnus to sample Lamp luminosity every five seconds",
  "subject": {
    "entities": [
      {
        "idPattern": "Lamp.*"
      }
    ],
    "condition": {
      "attrs": [
        "luminosity"
      ]
    }
  },
  "notification": {
    "http": {
      "url": "http://cygnus:5050/notify"
    },
    "attrs": [
      "luminosity"
    ],
    "attrsFormat": "legacy"
  },
  "throttling": 5
}'
```

<a name="formal-mode---time-series-data-queries"></a>
## *正規*モード - 時系列データのクエリ

データベースからデータを読み込む場合、*最小*と*正規*のモードに違いはありません。このチュートリアルの前のセクションを参照して、**STH-Comet** から時系列データをリクエストしてください

<a name="accessing-time-series-data-programmatically"></a>
# プログラミングによる時系列データへのアクセス

指定された時系列の JSON レスポンスが取得されると、生のデータを表示することはエンドユーザにとってほとんど役に立ちません。これは、棒グラフ、折れ線グラフ、またはテーブルリストに表示するために操作をする必要があります。これはグラフィカル・ツールではないため、**STH-Comet** の範疇ではありませんが、[Wirecloud](https://catalogue.fiware.org/enablers/application-mashup-wirecloud) や [Knowage](https://catalogue-server.fiware.org/enablers/data-visualization-knowage) などのマッシュアップやダッシュボード・コンポーネントに委任できます。

また、コーディング環境に適したサードパーティのグラフ作成ツールを使用して、検索して表示することもできます。たとえば、[chartjs](http://www.chartjs.org/) です。これの例は [Git Repository](https://github.com/Fiware/tutorials.Short-Term-History/blob/master/proxy/controllers/history.js) の `history` コントローラ内にあります。

基本的な処理は、検索と属性マッピングの2つのステップで構成されています。サンプルコードは以下のとおりです :

```javascript
function readLampLuminosity(id, aggMethod) {
    return new Promise(function(resolve, reject) {
      const url ="http://sth-comet:8666/STH/v1/contextEntities/type/Lamp/id/Lamp:001/attributes/luminosity"
      const options = { method: 'GET',
        url: url,
        qs: { aggrMethod: aggMethod, aggrPeriod: 'minute' },
        headers: 
         { 
           'fiware-servicepath': '/',
           'fiware-service': 'openiot' } };

    request(options, (error, response, body) => {
        return error ? reject(error) : resolve(JSON.parse(body));
    });
  });
}
```

```javascript
function cometToTimeSeries(cometResponse, aggMethod){

    const data = [];
    const labels = [];

    const values =  cometResponse.contextResponses[0].contextElement.attributes[0].values[0];
      let date = moment(values._id.origin);
    
      _.forEach(values.points, element => {
          data.push({ t: date.valueOf(), y: element[aggMethod] });
          labels.push(date.format( 'HH:mm'));
          date = date.clone().add(1, 'm');
      });

  return {
    labels,
    data
  };
}
```

変更されたデータは、フロントエンドに渡され、サードパーティのグラフ作成ツールによって処理されます。結果は次のとおりです : `http://localhost:3000/device/history/urn:ngsi-ld:Store:001`

<a name="next-steps"></a>
# 次のステップ

高度な機能を追加することで、アプリケーションに複雑さを加える方法を知りたいですか？このシリーズの他のチュートリアルを読むことで見つけることができます :

&nbsp; 101. [Getting Started](https://github.com/Fiware/tutorials.Getting-Started)<br/>
&nbsp; 102. [Entity Relationships](https://github.com/Fiware/tutorials.Entity-Relationships)<br/>
&nbsp; 103. [CRUD Operations](https://github.com/Fiware/tutorials.CRUD-Operations)<br/>
&nbsp; 104. [Context Providers](https://github.com/Fiware/tutorials.Context-Providers)<br/>
&nbsp; 105. [Altering the Context Programmatically](https://github.com/Fiware/tutorials.Accessing-Context)<br/> 
&nbsp; 106. [Subscribing to Changes in Context](https://github.com/Fiware/tutorials.Subscriptions)<br/>

&nbsp; 201. [Introduction to IoT Sensors](https://github.com/Fiware/tutorials.IoT-Sensors)<br/>
&nbsp; 202. [Provisioning an IoT Agent](https://github.com/Fiware/tutorials.IoT-Agent)<br/>
&nbsp; 203. [IoT over MQTT](https://github.com/Fiware/tutorials.IoT-over-MQTT)<br/>
&nbsp; 250. [Introduction to Fast-RTPS and Micro-RTPS ](https://github.com/Fiware/tutorials.Fast-RTPS-Micro-RTPS)<br/>

&nbsp; 301. [Persisting Context Data (Mongo-DB, MySQL, PostgreSQL)](https://github.com/Fiware/tutorials.Historic-Context)<br/>
&nbsp; 302. [Querying Time Series Data (Mongo-DB)](https://github.com/Fiware/tutorials.Short-Term-History)<br/>
&nbsp; 303. [Querying Time Series Data (Crate-DB)](https://github.com/Fiware/tutorials.Time-Series-Data)<br/>
