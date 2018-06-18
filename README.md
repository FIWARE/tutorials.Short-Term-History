![FIWARE Banner](https://fiware.github.io/tutorials.Historic-Context/img/fiware.png)

This tutorial is an introduction to [FIWARE STH-Comet](https://fiware-sth-comet.readthedocs.io/) - a generic enabler which is used to retrieve trend data from a Mongo-DB database. The tutorial activates the IoT sensors connected in the [previous tutorial](https://github.com/Fiware/tutorials.IoT-Agent) and persists measurements from those sensors into a database and retrieves time-based aggregations of that data.

The tutorial uses [cUrl](https://ec.haxx.se/) commands throughout, but is also available as [Postman documentation](http://fiware.github.io/tutorials.Historic-Context/)

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/4824d3171f823935dcab)


# Contents

TBC ...


# Persisting Time Series Data

> "The *"moment"* has no yesterday or tomorrow. It is not the result of thought and therefore has no time."
>
> — Bruce Lee


Within the FIWARE platform, historical context data can be persisted to a database using a combination of the **Orion 
Context Broker** and the **Cygnus** generic enabler. This results in a series of data points being written to the
database of your choice. Each time-stamped data point represents the state of context entities at a given moment in time.
The individual data points are relatively meaningless on their own, it is only through combining a series data points
that meaningful statistics such as maxima, minima and trends can be observed.

The creation and analysis of trend data is a common requirement of context-driven systems - therefore the FIWARE platform 
offers a generic enabler ([STH-Comet](https://fiware-sth-comet.readthedocs.io/)) specifically to deal with the issue of persisting and interpreting time series data. **STH-Comet** itself can be used in two modes:

* In *minimal* mode, **STH-Comet** is responsible for both data collection and interpreting the data when requested
* In *formal* mode, the collection of data is delegated to **Cygnus**, **STH-Comet** merely reads from an existing database.

Of the two modes of operation, the *formal* mode is more flexible, but *minimal* mode is simpler and easier to set-up. The key differences between the two are summarized in the table below:


|                                                         | *minimal* mode                | *formal* mode              |
|---------------------------------------------------------|-------------------------------|----------------------------|
| Is the system easy to set-up  properly?                 | Only one configuration supported - Easy to set up                | Highly configurable - Complex to set up          |
| Which component is responsible for a data persistance?  | **STH-Comet**                 | **Cygnus**                 |
| What is the role of **STH-Comet**?                      | Reading and writing data      | Data Read only             |
| What is the role of **Cygnus**?                         | Not Used                      | Data Write only            |
| Where is the data aggregated?                           | Mongo-DB database connected to **STH-Comet** only| Mongo-DB database connected to both **Cygnus** and **STH-Comet** |
| Can the system be configured to use other databases?    | No                            | Yes                        |
| Does the solution scale easily?                         | Does not scale easily - use for simple systems | Scales easily - use for complex systems |
| Can the system cope with high rates of throughput?      | No - use where throughput is low | Yes - use where throughput is high |


## Analyzing time series data

The appropriate use of time series data analysis will depend on your use case and the reliability of the data measurements you receive. Time series data analysis can be used to answer questions such as:

* What was the maximum measurement of a device within a given time period?
* What was the average measurement of a device within a given time period?
* What was the sum of the measurements sent by a device within a given time period?

It can also be used to reduce the signficance of each individual data point to exclude outliers by smoothing.



#### Device Monitor

For the purpose of this tutorial, a series of dummy IoT devices have been created, which will be attached to the context broker. Details of the architecture and protocol used can be found in the [IoT Sensors tutorial](https://github.com/Fiware/tutorials.IoT-Sensors).
The state of each device can be seen on the UltraLight device monitor web-page found at: `http://localhost:3000/device/monitor`

![FIWARE Monitor](https://fiware.github.io/tutorials.Short-Term-History/img/device-monitor.png)



# Architecture

This application builds on the components and dummy IoT devices created in 
[previous tutorials](https://github.com/Fiware/tutorials.IoT-Agent/). It will use three or four FIWARE components depending on the configuration of the system:
the [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/), the
[IoT Agent for Ultralight 2.0](http://fiware-iotagent-ul.readthedocs.io/en/latest/),
[STH-Comet](http://fiware-cygnus.readthedocs.io/en/latest/) and
[Cygnus](http://fiware-cygnus.readthedocs.io/en/latest/). 

Therefore the overall architecture will consist of the following elements:

* Four **FIWARE Generic Enablers**:
  * The FIWARE [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) which will receive requests using [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2)
  * The FIWARE [IoT Agent for Ultralight 2.0](http://fiware-iotagent-ul.readthedocs.io/en/latest/) which will receive northbound measurements from the dummy IoT devices in [Ultralight 2.0](http://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual) format and convert them to [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2) requests for the context broker to alter the state of the context entities
  * FIWARE [STH-Comet](http://fiware-sth-comet.readthedocs.io/) will:
    + interpret time-based data queries
    + subscribe to context changes and persist them into a **Mongo-DB** database  (*minimal* mode only)
   * FIWARE [Cygnus](http://fiware-cygnus.readthedocs.io/en/latest/) where it will subscribe to context changes and persist them into a **Mongo-DB** database (*formal* mode only) 

> :information_source: **Note:** **Cygnus** will only be used if **STH-Comet** is configured in *formal* mode.

* A [MongoDB](https://www.mongodb.com/) database:
  * Used by the **Orion Context Broker** to hold context data information such as data entities, subscriptions and registrations
  * Used by the **IoT Agent** to hold device information such as device URLs and Keys
  * Used as a data sink to hold time-based historical context data
     + In *minimal* mode - this is read and populated by **STH-Comet** 
     + In *formal* mode - this is populated by **Cygnus**  and read by **STH-Comet** 
* Three **Context Providers**:
  * The **Stock Management Frontend** is not used in this tutorial. It does the following:
    + Display store information and allow users to interact with the dummy IoT devices
    + Show which products can be bought at each store
    + Allow users to "buy" products and reduce the stock count.
  * A webserver acting as set of [dummy IoT devices](https://github.com/Fiware/tutorials.IoT-Sensors) using the [Ultralight 2.0](http://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual) protocol running over HTTP.
  * The **Context Provider NGSI** proxy is not used in this tutorial. It does the following:
    + receive requests using [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2)
    + makes requests to publicly available data sources using their own APIs in a proprietary format 
    + returns context data back to the Orion Context Broker in [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2) format.




Since all interactions between the elements are initiated by HTTP requests, the entities can be containerized and run from exposed ports. 

The specific architecture of both the *minimal* and *formal* configurations is discussed below.



# Prerequisites

## Docker and Docker Compose 

To keep things simple all components will be run using [Docker](https://www.docker.com). **Docker** is a container technology which allows to different components isolated into their respective environments. 

* To install Docker on Windows follow the instructions [here](https://docs.docker.com/docker-for-windows/)
* To install Docker on Mac follow the instructions [here](https://docs.docker.com/docker-for-mac/)
* To install Docker on Linux follow the instructions [here](https://docs.docker.com/install/)

**Docker Compose** is a tool for defining and running multi-container Docker applications. A  series of [YAML files](https://raw.githubusercontent.com/Fiware/tutorials.Historic-Context/master/cygnus) are used configure the required
services for the application. This means all container services can be brought up in a single command. Docker Compose is installed by default as part of Docker for Windows and  Docker for Mac, however Linux users will need to follow the instructions found [here](https://docs.docker.com/compose/install/)

## Cygwin for Windows

We will start up our services using a simple Bash script. Windows users should download [cygwin](http://www.cygwin.com/) to provide a command line functionality similar to a Linux distribution on Windows.




# Start Up

Before you start you should ensure that you have obtained or built the necessary Docker images locally. Please run

```console
./services create
``` 

>**Note** The `context-provider` image has not yet been pushed to Docker hub.
> Failing to build the Docker sources before proceeding will result in the following error:
>
>```
>Pulling context-provider (fiware/cp-web-app:latest)...
>ERROR: The image for the service you're trying to recreate has been removed.
>```


Thereafter, all services can be initialized from the command line by running the [services](https://github.com/Fiware/tutorials.Historic-Context/blob/master/services) Bash script provided within the repository:

```console
./services <command>
``` 

Where `<command>` will vary depending upon the mode we wish to activate.
This command will also import seed data from the previous tutorials and provision the dummy IoT sensors on startup.

>:information_source: **Note:** If you want to clean up and start over again you can do so with the following command:
>
>```console
>./services stop
>``` 
>



# Next Steps

Want to learn how to add more complexity to your application by adding advanced features?
You can find out by reading the other tutorials in this series:

&nbsp; 101. [Getting Started](https://github.com/Fiware/tutorials.Getting-Started)<br/>
&nbsp; 102. [Entity Relationships](https://github.com/Fiware/tutorials.Entity-Relationships)<br/>
&nbsp; 103. [CRUD Operations](https://github.com/Fiware/tutorials.CRUD-Operations)<br/>
&nbsp; 104. [Context Providers](https://github.com/Fiware/tutorials.Context-Providers)<br/>
&nbsp; 105. [Altering the Context Programmatically](https://github.com/Fiware/tutorials.Accessing-Context)<br/> 
&nbsp; 106. [Subscribing to Changes in Context](https://github.com/Fiware/tutorials.Subscriptions)<br/>

&nbsp; 201. [Introduction to IoT Sensors](https://github.com/Fiware/tutorials.IoT-Sensors)<br/>
&nbsp; 202. [Provisioning an IoT Agent](https://github.com/Fiware/tutorials.IoT-Agent)<br/>

&nbsp; 301. [Persisting Context Data](https://github.com/Fiware/tutorials.Historic-Context)<br/>
&nbsp; 302. [Persisting Time Series Data](https://github.com/Fiware/tutorials.Short-Term-History)<br/>
