# Temporal Interfaces[<img src="https://img.shields.io/badge/NGSI-LD-d6604d.svg" width="90"  align="left" />](https://www.etsi.org/deliver/etsi_gs/CIM/001_099/009/01.04.01_60/gs_cim009v010401p.pdf)[<img src="https://fiware.github.io/tutorials.Short-Term-History/img/fiware.png" align="left" width="162">](https://www.fiware.org/)<br/>

[![FIWARE Core Context Management](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/core.svg)](https://github.com/FIWARE/catalogue/blob/master/core/README.md)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.Short-Term-History.svg)](https://opensource.org/licenses/MIT)
[![Support badge](https://img.shields.io/badge/tag-fiware-orange.svg?logo=stackoverflow)](https://stackoverflow.com/questions/tagged/fiware)
[![JSON LD](https://img.shields.io/badge/JSON--LD-1.1-f06f38.svg)](https://w3c.github.io/json-ld-syntax/) <br/>
[![Documentation](https://img.shields.io/readthedocs/ngsi-ld-tutorials.svg)](https://ngsi-ld-tutorials.rtfd.io)

This tutorial is an introduction to the temporal interface of NGSI-LD, an **optional** add-on to context broker implementations. The tutorial activates the IoT animal collars connected in the
[previous tutorial](https://github.com/FIWARE/tutorials.IoT-Agent/tree/NGSI-LD) and persists measurements from those sensors into a
database and retrieves time-based aggregations of that data.

The tutorial uses [cUrl](https://ec.haxx.se/) commands throughout, but is also available as
[Postman documentation](https://fiware.github.io/tutorials.Short-Term-History/ngsi-ld.html)

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/4824d3171f823935dcab)。

## Contents

<details>
<summary><strong>Details</strong></summary>


</details>

# Querying Temporal Data

> "I could dance with you till the cows come home. Better still, I'll dance with the cows and you come home."
>
> — Groucho Marx (Duck Soup)

NGSI-LD introduces a standardized mechanism for persisting and retrieving historical context data. Conventionally, context brokers only deal with current context - they have no memory, however NGSI-LD context brokers can be extended to offer historical context data in a variety of JSON based formats. This additional functionality comes at a cost however, and for performance reasons, may not be available by default.

Enabled Context brokers can persist historic context using the database of their choice. The NGSI-LD temporal interface is agnostic to the actual persistence mechanism to be used by the context broker - the interface merely specifies the outputs required when various queries take place. Furthermore NGSI-LD also specifies a mechanism for amending values of historic context using the `instanceId` attribute.

The result is a series of data points timestamped using the `observedAt` _property-of-a-property_. Each time-stamped data point represents the state of context entities at a given moment in time. The individual data points are relatively meaningless on their own, it is only through combining a
series data points that meaningful statistics such as maxima, minima and trends can be observed.

The creation and analysis of trend data is a common requirement of context-driven systems. Within FIWARE, there are two common paradigms in use - either activating the temporal interface or subscribing to individual context entities and persisting them into a time-series database (using a component such as QuantumLeap) - the latter is described in a [separate tutorial](https://github.com/FIWARE/tutorials.IoT-Agent/tree/NGSI-LD).

Which mechanism to use should be it should be borne in mind when architecting such a system. The advantage of using a subscription mechanism is that only the subscribed entities are persisted, saving disk space. The advantage of the temporal interface is that it is provided by the context broker directly - no subscriptions are needed and HTTP traffic is reduced. Furthermore, the temporal interface can be queried across all context entities, not merely those which satisfy a subscription.

#### Device Monitor

For the purpose of this tutorial, a series of dummy animal collar IoT devices have been created, which will be attached to the context
broker. Details of the architecture and protocol used can be found in the
[IoT Sensors tutorial](https://github.com/FIWARE/tutorials.IoT-Sensors/tree/NGSI-LD). The state of each device can be
seen on the UltraLight device monitor web page found at: `http://localhost:3000/device/monitor`

![FIWARE Monitor](https://fiware.github.io/tutorials.Time-Series-Data/img/farm-devices.png)

# Architecture

This application builds on the components and dummy IoT devices created in
[previous tutorials](https://github.com/FIWARE/tutorials.IoT-Agent/tree/NGSI-LD). It will use two FIWARE components: the
[Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) and the
[IoT Agent for Ultralight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/). In addition the optional temporal interface is serviced using an add-on called **Mintaka**.

Therefore the overall architecture will consist of the following elements:

-   The [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) which will receive requests using
    [NGSI-LD](https://forge.etsi.org/swagger/ui/?url=https://forge.etsi.org/rep/NGSI-LD/NGSI-LD/raw/master/spec/updated/generated/full_api.json)
-   The FIWARE [IoT Agent for UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/) which will receive
    northbound device measures requests using [UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual) syntax and convert them to
    [NGSI-LD](https://forge.etsi.org/swagger/ui/?url=https://forge.etsi.org/rep/NGSI-LD/NGSI-LD/raw/master/spec/updated/generated/full_api.json)
-   The underlying [MongoDB](https://www.mongodb.com/) database :
    -   Used by the **Orion Context Broker** to hold context data information such as data entities, subscriptions and
        registrations
    -   Used by the **IoT Agent** to hold device information such as device URLs and Keys
-   A [Timescale](https://www.timescale.com/) timeseries database for persisting historic context.
-   The **Mintaka** add-on which services the temporal interface and is also responsible for persisting the context
-   The **Tutorial Application** does the following:
    -   Offers static `@context` files defining the context entities within the system.
    -   Acts as set of dummy [agricultural IoT devices](https://github.com/FIWARE/tutorials.IoT-Sensors/tree/NGSI-LD)
        using the
        [UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual)
        protocol running over HTTP.

Since all interactions between the elements are initiated by HTTP requests, the entities can be containerized and run
from exposed ports.

![](https://fiware.github.io/tutorials.IoT-Agent/img/architecture-ld.png)

# Prerequisites

## Docker and Docker Compose

To keep things simple all components will be run using [Docker](https://www.docker.com). **Docker** is a container
technology which allows to different components isolated into their respective environments.

-   To install Docker on Windows follow the instructions [here](https://docs.docker.com/docker-for-windows/)
-   To install Docker on Mac follow the instructions [here](https://docs.docker.com/docker-for-mac/)
-   To install Docker on Linux follow the instructions [here](https://docs.docker.com/install/)

**Docker Compose** is a tool for defining and running multi-container Docker applications. A series of
[YAML files](https://github.com/FIWARE/tutorials.Short-Term-History/tree/master/docker-compose) are used configure the
required services for the application. This means all container services can be brought up in a single command. Docker
Compose is installed by default as part of Docker for Windows and Docker for Mac, however Linux users will need to
follow the instructions found [here](https://docs.docker.com/compose/install/)

You can check your current **Docker** and **Docker Compose** versions using the following commands:

```console
docker-compose -v
docker version
```

Please ensure that you are using Docker version 18.03 or higher and Docker Compose 1.21 or higher and upgrade if
necessary.

## Cygwin for Windows

We will start up our services using a simple Bash script. Windows users should download [cygwin](http://www.cygwin.com/)
to provide a command-line functionality similar to a Linux distribution on Windows.

# Start Up

Before you start you should ensure that you have obtained or built the necessary Docker images locally. Please clone the
repository and create the necessary images by running the commands as shown:

```console
git clone https://github.com/FIWARE/tutorials.Short-Term-History.git
cd tutorials.Short-Term-History
git checkout NGSI-LD

./services create
```

Thereafter, all services can be initialized from the command-line by running the
[services](https://github.com/FIWARE/tutorials.Short-Term-History/blob/NGSI-LD/services) Bash script provided within the
repository:

```console
./services <command>
```

Where `<command>` will vary depending upon the mode we wish to activate. This command will also import seed data from
the previous tutorials and provision the dummy IoT sensors on startup.

> :information_source: **Note:** If you want to clean up and start over again you can do so with the following command:
>
> ```console
> ./services stop
> ```

# Mintaka mode (STH-Comet only)

In the _minimal_ configuration, **STH-Comet** is used to persisting historic context data and also used to make
time-based queries. All operations take place on the same port `8666`. The MongoDB instance listening on the standard
`27017` port is used to hold data the historic context data as well as holding data related to the **Orion Context
Broker** and the **IoT Agent**. The overall architecture can be seen below:

![](https://fiware.github.io/tutorials.Short-Term-History/img/sth-comet.png)

## Minitaka Configuration

```yaml
mintaka:
    image: fiware/mintaka:${MINTAKA_VERSION}
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

## Orion Configuration

```yaml
 orion:
    image: fiware/orion-ld:${ORION_LD_VERSION}
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

The `sth-comet` container is listening on one port:

-   The Operations for port for STH-Comet - `8666` is where the service will be listening for notifications from the
    Orion context broker as well as time based query requests from cUrl or Postman

The `sth-comet` container is driven by environment variables as shown:

| Key          | Value            | Description                                                                                                    |
| ------------ | ---------------- | -------------------------------------------------------------------------------------------------------------- |
| STH_HOST     | `0.0.0.0`        | The address where STH-Comet is hosted - within this container it means all IPv4 addresses on the local machine |
| STH_PORT     | `8666`           | Operations Port that STH-Comet listens on, it is also used when subscribing to context data changes            |
| DB_PREFIX    | `sth_`           | The prefix added to each database entity if none is provided                                                   |
| DB_URI       | `mongo-db:27017` | The MongoDB server which STH-Comet will contact to persist historical context data                             |
| LOGOPS_LEVEL | `DEBUG`          | The logging level for STH-Comet                                                                                |

## _minimal_ mode - Start up

To start the system using the _minimal_ configuration using **STH-Comet** only, run the following command:

```console
./services sth-comet
```

### STH-Comet - Checking Service Health

Once STH-Comet is running, you can check the status by making an HTTP request to the exposed `STH_PORT` port. If the
response is blank, this is usually because **STH-Comet** is not running or is listening on another port.

#### :one: Request:

```console
curl -X GET \
  'http://localhost:8666/version'
```

#### Response:

The response will look similar to the following:

```json
{
    "version": "2.3.0-next"
}
```

> **Troubleshooting:** What if the response is blank ?
>
> -   To check that a docker container is running try
>
> ```bash
> docker ps
> ```
>
> You should see several containers running. If `sth-comet` or `cygnus` is not running, you can restart the containers
> as necessary.

### Generating Context Data

For the purpose of this tutorial, we must be monitoring a system where the context is periodically being updated. The
dummy IoT Sensors can be used to do this. Open the device monitor page at `http://localhost:3000/device/monitor` and
unlock a **Smart Door** and switch on a **Smart Lamp**. This can be done by selecting an appropriate the command from
the drop down list and pressing the `send` button. The stream of measurements coming from the devices can then be seen
on the same page:

![](https://fiware.github.io/tutorials.Short-Term-History/img/door-open.gif)

## _minimal_ mode - Subscribing STH-Comet to Context Changes

Once a dynamic context system is up and running, under minimal mode, **STH-Comet** needs to be informed of changes in
context. Therefore we need to set up a subscription in the **Orion Context Broker** to notify **STH-Comet** of these
changes. The details of the subscription will differ dependent upon the device being monitored and the sampling rate.

### STH-Comet - Aggregate Motion Sensor Count Events

The rate of change of the **Motion Sensor** is driven by events in the real-world. We need to receive every event to be
able to aggregate the results.

This is done by making a POST request to the `/v2/subscription` endpoint of the **Orion Context Broker**.

-   The `fiware-service` and `fiware-servicepath` headers are used to filter the subscription to only listen to
    measurements from the attached IoT Sensors
-   The `idPattern` in the request body ensures that **STH-Comet** will be informed of all **Motion Sensor** data
    changes.
-   The notification `url` must match the configured `STH_PORT`
-   The `attrsFormat=legacy` is required since **STH-Comet** currently only accepts notifications in the older NGSI v1
    format.

#### :two: Request:

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

### STH-Comet - Sample Lamp Luminosity

The luminosity of the **Smart Lamp** is constantly changing, we only need to **sample** the values to be able to work
out relevant statistics such as minimum and maximum values and rates of change.

This is done by making a POST request to the `/v2/subscription` endpoint of the **Orion Context Broker** and including
the `throttling` attribute in the request body.

-   The `fiware-service` and `fiware-servicepath` headers are used to filter the subscription to only listen to
    measurements from the attached IoT Sensors
-   The `idPattern` in the request body ensures that **STH-Comet** will be informed of all **Smart Lamp** data changes
    only
-   The notification `url` must match the configured `STH_PORT`
-   The `attrsFormat=legacy` is required since **STH-Comet** currently only accepts notifications in the older NGSI v1
    format.
-   The `throttling` value defines the rate that changes are sampled.

> :information_source: **Note:** Be careful when throttling subscriptions as sequential updates will not be persisted as
> expected.
>
> For example if an UltraLight device sends the measurement `t|20|l|1200` it will be a single atomic commit and both
> attributes will be included the notification to **STH-Comet** however is a device sends `t|20#l|1200` this will be
> treated as two atomic commits - a notification will be sent for the first change in `t`, but the second change in `l`
> will be ignored as the entity has been recently updated within the sampling period.

#### :three: Request:

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
    ]
  },
  "throttling": 5
}'
```

# Time Series Data Queries

The queries in this section assume you have already connected **STH-Comet** using either _minimal_ mode or _formal_ mode
and have collected some data.

## Prerequisites

**STH-Comet** will only be able to retrieve time series data if sufficient data points have already been aggregated
within the system. Please ensure that the **Smart Door** has been unlocked and the **Smart Lamp** has been switched on
and the subscriptions have been registered. Data should be collected for at least a minute before the tutorial.

### Check that Subscriptions Exist

You can note that the `fiware-service` and `fiware-servicepath` headers must be set in the query and match the values
used when setting up the subscription

#### :four: Request:

```console
curl -X GET \
  'http://localhost:1026/v2/subscriptions/' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```

#### Response:

```json
[
    {
        "id": "5b39e3c11615e2e55a8df103",
        "description": "Notify STH-Comet of all Motion Sensor count changes",
        "status": "active",
        "subject": { ...ETC },
        "notification": {
            "timesSent": 6,
            "lastNotification": "2018-07-02T08:36:04.00Z",
            "attrs": ["count"],
            "attrsFormat": "legacy",
            "http": { "url": "http://sth-comet:8666/notify" },
            "lastSuccess": "2018-07-02T08:36:04.00Z"
        }
    },
    {
        "id": "5b39e3c31615e2e55a8df104",
        "description": "Notify STH-Comet to sample Lamp changes every five seconds",
        "status": "active",
        "subject": { ...ETC },
        "notification": {
            "timesSent": 4,
            "lastNotification": "2018-07-02T08:36:00.00Z",
            "attrs": ["luminosity"],
            "attrsFormat": "legacy",
            "http": { "url": "http://sth-comet:8666/notify" },
            "lastSuccess": "2018-07-02T08:36:01.00Z"
        },
        "throttling": 5
    }
]
```

The result should not be empty. Within the `notification` section of the response, you can see several additional
`attributes` which describe the health of each subscription.

If the criteria of the subscription have been met, `timesSent` should be greater than `0`. A zero value would indicate
that the `subject` of the subscription is incorrect or the subscription has created with the wrong `fiware-service-path`
or `fiware-service` header

The `lastNotification` should be a recent timestamp - if this is not the case, then the devices are not regularly
sending data. Remember to unlock the **Smart Door** and switch on the **Smart Lamp**

The `lastSuccess` should match the `lastNotification` date - if this is not the case then **Cygnus** or **STH Comet**
are not receiving the subscription properly. Check that the hostname and port are correct.

Finally, check that the `status` of the subscription is `active` - an expired subscription will not fire.

## Offsets, Limits and Pagination

### List the first N sampled values

This example shows the first 3 sampled `luminosity` values from `Lamp:001`.

To obtain the short term history of a context entity attribute, send a GET request to
`../STH/v1/contextEntities/type/<Entity>/id/<entity-id>/attributes/<attribute>`

the `hLimit` parameter restricts the result to N values. `hOffset=0` will start with the first value.

#### :five: Request:

```console
curl -X GET \
  'http://localhost:8666/STH/v1/contextEntities/type/Lamp/id/Lamp:001/attributes/luminosity?hLimit=3&hOffset=0' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```

#### Response:

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

### List N sampled values at an Offset

This example shows the fourth, fifth and sixth sampled `count` values from `Motion:001`.

To obtain the short term history of a context entity attribute, send a GET request to
`../STH/v1/contextEntities/type/<Entity>/id/<entity-id>/attributes/<attribute>`

the `hLimit` parameter restricts the result to N values. Setting `hOffset` to a non-zero value will start from the Nth
measurement

#### :six: Request:

```console
curl -X GET \
  'http://localhost:8666/STH/v1/contextEntities/type/Motion/id/Motion:001/attributes/count?hLimit=3&hOffset=3' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```

#### Response:

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

### List the latest N sampled values

This example shows latest three sampled `count` values from `Motion:001`.

To obtain the short term history of a context entity attribute, send a GET request to
`../STH/v1/contextEntities/type/<Entity>/id/<entity-id>/attributes/<attribute>`

If the `lastN` parameter is set, the result will return the N latest measurements only.

#### :seven: Request:

```console
curl -X GET \
  'http://localhost:8666/STH/v1/contextEntities/type/Motion/id/Motion:001/attributes/count?lastN=3' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```

#### Response:

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

## Time Period Queries

### List the sum of values over a time period

This example shows total `count` values from `Motion:001` over each minute

To obtain the short term history of a context entity attribute, send a GET request to
`../STH/v1/contextEntities/type/<Entity>/id/<entity-id>/attributes/<attribute>`

The `aggrMethod` parameter determines the type of aggregation to perform over the time series, the `aggrPeriod` is one
of `second`, `minute`, `hour` or `day`.

Always select the most appropriate time period based on the frequency of your data collection. `minute` has been
selected because the `Motion:001` is firing a few times within each minute.

#### :eight: Request:

```console
curl -X GET \
  'http://localhost:8666/STH/v1/contextEntities/type/Motion/id/Motion:001/attributes/count?aggrMethod=sum&aggrPeriod=minute' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```

#### Response:

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
                                    },
                                    ...etc
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

Querying for the mean value within a time period is not directly supported.

This example shows sum of `luminosity` values from `Lamp:001` over each minute. When combined with the number of samples
the within the time period an average can be calculated from the data.

To obtain the short term history of a context entity attribute, send a GET request to
`../STH/v1/contextEntities/type/<Entity>/id/<entity-id>/attributes/<attribute>`

The `aggrMethod` parameter determines the type of aggregation to perform over the time series, the `aggrPeriod` is one
of `second`, `minute`, `hour` or `day`.

#### :nine: Request:

```console
curl -X GET \
  'http://localhost:8666/STH/v1/contextEntities/type/Lamp/id/Lamp:001/attributes/luminosity?aggrMethod=sum&aggrPeriod=minute' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```

#### Response:

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
                                    },
                                    ...etc
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

### List the minimum of a value over a time period

This example shows minimum `luminosity` values from `Lamp:001` over each minute

To obtain the short term history of a context entity attribute, send a GET request to
`../STH/v1/contextEntities/type/<Entity>/id/<entity-id>/attributes/<attribute>`

The `aggrMethod` parameter determines the type of aggregation to perform over the time series, the `aggrPeriod` is one
of `second`, `minute`, `hour` or `day`.

The luminocity of the **Smart Lamp** is continually changing and therefore tracking the minimum value makes sense. The
**Motion Sensor** is not suitable for this as it only offers binary values.

#### :one::zero: Request:

```console
curl -X GET \
  'http://localhost:8666/STH/v1/contextEntities/type/Lamp/id/Lamp:001/attributes/luminosity?aggrMethod=min&aggrPeriod=minute' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```

#### Response:

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

### List the maximum of a value over a time period

This example shows maximum `luminosity` values from `Lamp:001` over each minute

To obtain the short term history of a context entity attribute, send a GET request to
`../STH/v1/contextEntities/type/<Entity>/id/<entity-id>/attributes/<attribute>`

The `aggrMethod` parameter determines the type of aggregation to perform over the time series, the `aggrPeriod` is one
of `second`, `minute`, `hour` or `day`.

#### :one::one: Request:

```console
curl -X GET \
  'http://localhost:8666/STH/v1/contextEntities/type/Lamp/id/Lamp:001/attributes/luminosity?aggrMethod=max&aggrPeriod=minute' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```

#### Response:

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
                                    },
                                    ...etc
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

# _formal_ mode (Cygnus + STH-Comet)

The _formal_ configuration is uses **Cygnus** to persist historic context data into a MongoDB database in the same
manner as had been presented in the [previous tutorial](https://github.com/FIWARE/tutorials.Historic-Context-Flume). The
existing MongoDB instance (listening on the standard `27017` port) is used to hold data related to the **Orion Context
Broker**, the **IoT Agent** and the historic context data persisted by **Cygnus**. **STH-Comet** is also attached to the
same database to read data from it. The overall architecture can be seen below:

![](https://fiware.github.io/tutorials.Short-Term-History/img/cygnus-sth-comet.png)

## Database Server Configuration

```yaml
mongo-db:
    image: mongo:4.2
    hostname: mongo-db
    container_name: db-mongo
    ports:
        - "27017:27017"
    networks:
        - default
```

## STH-Comet Configuration

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

## Cygnus Configuration

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

The `sth-comet` container is listening on one port:

-   The Operations for port for STH-Comet - `8666` is where the service will be listening for time based query requests
    from cUrl or Postman

The `sth-comet` container is driven by environment variables as shown:

| Key          | Value            | Description                                                                                                    |
| ------------ | ---------------- | -------------------------------------------------------------------------------------------------------------- |
| STH_HOST     | `0.0.0.0`        | The address where STH-Comet is hosted - within this container it means all IPv4 addresses on the local machine |
| STH_PORT     | `8666`           | Operations Port that STH-Comet listens on                                                                      |
| DB_PREFIX    | `sth_`           | The prefix added to each database entity if none is provided                                                   |
| DB_URI       | `mongo-db:27017` | The MongoDB server which STH-Comet will contact to persist historical context data                             |
| LOGOPS_LEVEL | `DEBUG`          | The logging level for STH-Comet                                                                                |

The `cygnus` container is listening on two ports:

-   The Subscription Port for Cygnus - `5050` is where the service will be listening for notifications from the Orion
    context broker
-   The Management Port for Cygnus - `5080` is exposed purely for tutorial access - so that cUrl or Postman can make
    provisioning commands without being part of the same network.

The `cygnus` container is driven by environment variables as shown:

| Key                 | Value            | Description                                                                                          |
| ------------------- | ---------------- | ---------------------------------------------------------------------------------------------------- |
| CYGNUS_MONGO_HOSTS  | `mongo-db:27017` | Comma separated list of MongoDB servers which Cygnus will contact to persist historical context data |
| CYGNUS_LOG_LEVEL    | `DEBUG`          | The logging level for Cygnus                                                                         |
| CYGNUS_SERVICE_PORT | `5050`           | Notification Port that Cygnus listens when subscribing to context data changes                       |
| CYGNUS_API_PORT     | `5080`           | Port that Cygnus listens on for operational reasons                                                  |

## _formal_ mode - Start up

To start the system using the _formal_ configuration using **Cygnus** and **STH-Comet**, run the following command:

```console
./services cygnus
```

### STH-Comet - Checking Service Health

Once **STH-Comet** is running, you can check the status by making an HTTP request to the exposed `STH_PORT` port. If the
response is blank, this is usually because **STH-Comet** is not running or is listening on another port.

#### :one::two: Request:

```console
curl -X GET \
  'http://localhost:8666/version'
```

#### Response:

The response will look similar to the following:

```json
{
    "version": "2.3.0-next"
}
```

### Cygnus - Checking Service Health

Once **Cygnus** is running, you can check the status by making an HTTP request to the exposed `CYGNUS_API_PORT` port. If
the response is blank, this is usually because **Cygnus** is not running or is listening on another port.

#### :one::three: Request:

```console
curl -X GET \
  'http://localhost:5080/v1/version'
```

#### Response:

The response will look similar to the following:

```json
{
    "success": "true",
    "version": "1.8.0_SNAPSHOT.ed50706880829e97fd4cf926df434f1ef4fac147"
}
```

> **Troubleshooting:** What if either response is blank ?
>
> -   To check that a docker container is running try
>
> ```bash
> docker ps
> ```
>
> You should see several containers running. If `sth-comet` or `cygnus` is not running, you can restart the containers
> as necessary.

### Generating Context Data

For the purpose of this tutorial, we must be monitoring a system where the context is periodically being updated. The
dummy IoT Sensors can be used to do this. Open the device monitor page at `http://localhost:3000/device/monitor` and
unlock a **Smart Door** and switch on a **Smart Lamp**. This can be done by selecting an appropriate the command from
the drop down list and pressing the `send` button. The stream of measurements coming from the devices can then be seen
on the same page:

![](https://fiware.github.io/tutorials.Short-Term-History/img/door-open.gif)

## _formal_ mode - Subscribing Cygnus to Context Changes

In _formal_ mode, **Cygnus** is responsible for the persistence of historic context data. Once a dynamic context system
is up and running, we need to set up a subscription in the **Orion Context Broker** to notify **Cygnus** of changes in
context - **STH-Comet** will only be used to read the persisted data.

### Cygnus - Aggregate Motion Sensor Count Events

The rate of change of the **Motion Sensor** is driven by events in the real-world. We need to receive every event to be
able to aggregate the results.

This is done by making a POST request to the `/v2/subscription` endpoint of the **Orion Context Broker**.

-   The `fiware-service` and `fiware-servicepath` headers are used to filter the subscription to only listen to
    measurements from the attached IoT Sensors
-   The `idPattern` in the request body ensures that **Cygnus** will be informed of all **Motion Sensor** data changes.
-   The notification `url` must match the configured `CYGNUS_API_PORT`

#### :one::four: Request:

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
      "url": "http://cygnus:5051/notify"
    },
    "attrs": [
      "count"
    ]
  }
}'
```

### Cygnus - Sample Lamp Luminosity

The luminosity of the **Smart Lamp** is constantly changing, we only need to **sample** the values to be able to work
out relevant statistics such as minimum and maximum values and rates of change.

This is done by making a POST request to the `/v2/subscription` endpoint of the **Orion Context Broker** and including
the `throttling` attribute in the request body.

-   The `fiware-service` and `fiware-servicepath` headers are used to filter the subscription to only listen to
    measurements from the attached IoT Sensors
-   The `idPattern` in the request body ensures that **Cygnus** will be informed of all **Smart Lamp** data changes only
-   The notification `url` must match the configured `CYGNUS_API_PORT`
-   The `throttling` value defines the rate that changes are sampled.

#### :one::five: Request:

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
      "url": "http://cygnus:5051/notify"
    },
    "attrs": [
      "luminosity"
    ]
  },
  "throttling": 5
}'
```

## _formal_ mode - Time Series Data Queries

When reading data from the database, there is no difference between _minimal_ and _formal_ mode, please refer to the
previous section of this tutorial to request time-series data from **STH-Comet**

# Accessing Time Series Data Programmatically

Once the JSON response for a specified time series has been retrieved, displaying the raw data is of little use to an
end user. It must be manipulated to be displayed in a bar chart, line graph or table listing. This is not within the
domain of **STH-Comet** as it not a graphical tool, but can be delegated to a mashup or dashboard component such as
[Wirecloud](https://github.com/FIWARE/catalogue/blob/master/processing/README.md#Wirecloud) or
[Knowage](https://github.com/FIWARE/catalogue/blob/master/processing/README.md#Knowage)

It can also be retrieved and displayed using a third-party graphing tool appropriate to your coding environment - for
example [chartjs](http://www.chartjs.org/). An example of this can be found within the `history` controller in the
[Git Repository](https://github.com/FIWARE/tutorials.Step-by-Step/blob/master/context-provider/controllers/history.js)

The basic processing consists of two-step - retrieval and attribute mapping, sample code can be seen below:

```javascript
function readCometLampLuminosity(id, aggMethod) {
    return new Promise(function(resolve, reject) {
        const url = "http://sth-comet:8666/STH/v1/contextEntities/type/Lamp/id/Lamp:001/attributes/luminosity";
        const options = {
            method: "GET",
            url: url,
            qs: { aggrMethod: aggMethod, aggrPeriod: "minute" },
            headers: {
                "fiware-servicepath": "/",
                "fiware-service": "openiot"
            }
        };

        request(options, (error, response, body) => {
            return error ? reject(error) : resolve(JSON.parse(body));
        });
    });
}
```

```javascript
function cometToTimeSeries(cometResponse, aggMethod) {
    const data = [];
    const labels = [];

    const values = cometResponse.contextResponses[0].contextElement.attributes[0].values[0];
    let date = moment(values._id.origin);

    _.forEach(values.points, element => {
        data.push({ t: date.valueOf(), y: element[aggMethod] });
        labels.push(date.format("HH:mm"));
        date = date.clone().add(1, "m");
    });

    return {
        labels,
        data
    };
}
```

The modified data is then passed to the frontend to be processed by the third-party graphing tool. The result is shown
here: `http://localhost:3000/device/history/urn:ngsi-ld:Store:001`

# Next Steps

Want to learn how to add more complexity to your application by adding advanced features? You can find out by reading
the other [tutorials in this series](https://fiware-tutorials.rtfd.io)

---

## License

[MIT](LICENSE) © 2018-2020 FIWARE Foundation e.V.
