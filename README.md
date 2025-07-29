# The way to RPC 2.0

Since epoch 104 the Qubic integration layer provides easy access to the Qubic network. It allows sending transactions, querying smart contracts and provides information like transaction and tick data.

The integration layer is used by 3rd party applications, like explorers, wallets and exchanges. The advantage of this layer is that archived data is easily accessible and users do not have to understand the network internas in detail and do not have to implement node communication.

## Integration APIs

The integration layer provides several different APIs. The most important ones are the live API and the archive API.

The live API talks to the Qubic nodes directly and provides most current data. It fetches information like qu balances and asset information and allows to send transactions and query smart contracts. The nodes provide a current view onto the data for the current epoch.

The data from the archive API instead provides archived data. The data gets collected from the nodes and finally stored in a database. Until recently all this was done by the qubic-archiver. 

## Original solution

The original archive implementation ([qubic-archiver](https://github.com/qubic/go-archiver)) is a fast, monolithic, and specialized solution. It processes the data for every tick, validates it by using quorum data, stores it in an embedded database (pebble), and serves it via GRPC and rest-like endpoints. Redundancy is achieved by running several instances in different data centers in parallel. While this solution still works perfectly fine it is not future proof enough to handle the expected growth long term.

To be sustainable long term certain challenges need to be solved. Some of the most important requirements are

* Load: be able to scale with the growing number of API requests.
* Data: be able to handle large amounts of data.
* Features: be able to satisfy more requirements.

### Load

Not long ago in beginning 2025 the whole integration layer needed to serve up to 10000 requests per minute. Only six months later this number has doubled and we serve more than 20000 request per minute at peak times. This load is expected to grow as the Qubic project faces more and more adaption.

To be able to scale as needed we decided to split the querying part into a separate application so that the archiver can focus on processing the data.

### Data

While one year ago the amount of all the data that we collect was a low three digit GB number in mid 2025 we have between 600 GB and 900 GB of data (depending on the compaction status in the database). A large part of this data is vote related quorum data. Event data is not included in this numbers.

As one can imagine it is not easy to handle such large monolithic blocks of data.

To be able to scale with the increasing storage requirements we decided to split the data into different parts and store it in a external database. The parts that are frequently accessed need to be available quickly and have flexible indexing while the parts that are only needed for archiving purpose can be stored in another way.

### Features

The original implementation focuses on performance. In principle the data is organized in a key value store that allows nearly instant access via the provided index. While this is fast, it is not very flexible and limits the possibilities of how to query the data. This, in turn, limits the possibilities for users of the API. Changes to the API require work-intensive changes to the access layer or even restructuring of the data.

To be able to provide the data in a more flexible and less work intensive way, we decided to use a solution that allows to index all the relevant data and that allows us to extend the indexing later on. Furthermore we decided to provide an API with a wide variety of filtering options.


## New solution

As already stated the solution to our challenges was in splitting the system into several parts that are specialized to the required task. For this we needed to

* Create databases for storing the data.
* Create ingestion pipelines for transporting the data.
* Create a service to provide the new query API.
* Handle the asynchronous nature of the new solution.
* Modify the archiver to focus on processing.

### Data storage

We have different kind of data that needs different ways of access and therefore we decided to use different ways of storing the data. There is

* Data that needs to be searchable and is frequently accessed.
* Data that is mainly stored for historical reasons and backup and not frequently accessed.
* Data that is temporarily stored for processing and short term backup/recovery (see later).

```
+------------------+  epoch data   +---------+
|     Archiver     | ------------> | Archive |
+------------------+               +---------+
  |
  | searchable data
  v
+------------------+
|  Search engine   |
+------------------+
```

#### Searchable data

One of the hardest decision was to find a suitable database for the data that needs to be served frequently and with high performance, like transactions, tick data and events. We did experiment with several solutions from relational databases to online cloud solutions but they were not able to handle the expected amount of data well. Finally we decided to go with a search engine and used [Elasticsearch](https://www.elastic.co/elasticsearch) that is based on [Apache Lucene](https://lucene.apache.org/) but offers useful features for clustering and replication. One elasticsearch cluster consists out of at least three nodes to provide enough redundancy (for example for maintenance).

The most important searchable data is transaction and tick data. Events are also available but not in production yet. For each kind of searchable data there is one search index within elasticsearch.

From a technical perspective a search `index` is defined by a search `template`. The template describes the index and if data is added it will be handled according to the template. To be able to later change the index we use `aliases` for every index. That way we can manage the data without the need to change the clients, for example, if we need to reindex the data with a new template version into another index later. So every index looks like this in general:

```
+-----------+     +-----------+     +--------------+
| xyz-alias | --> | xyz-index | <-- | xyz-template |
+-----------+     +-----------+     +--------------+
```

From the client persepective the data store looks like this:

```
  +--------------------+       +--------------+
  |       client       |   --> | elasticsarch |
  +--------------------+       +--------------+
..........................
: elasticsearch          :
:                        :
: +--------------------+ :
: |  computors-alias   | :
: +--------------------+ :
: +--------------------+ :
: |    events-alias    | :
: +--------------------+ :
: +--------------------+ :
: |  tick-data-alias   | :
: +--------------------+ :
: +--------------------+ :
: | transactions-alias | :
: +--------------------+ :
:                        :
:........................:
```

#### Archived data

For the data archive we decided to go with a file store where every epoch is stored as a data package. To make this possible we had to change the archiver to be able to split data per epoch and allow it to prune old data to keep the overall data size manageable. The per epoch data get's shipped to a server where it is archived. Third parties can get the data from there and import it. The archive is also useful as a backup.

```
+----------+  data per epoch   +------------+
| archiver | ----------------> |  archive   |
+----------+                   +------------+
                                 ^
                                 |
                                 |
+----------+                   +------------+
| clients  | ----------------> | web server |
+----------+                   +------------+
```

#### Backup and disaster recovery

To be able to restore the data in case of disaster there are several layers in place. First of all there are regular backups of the systems so that we can restore them in case something goes wrong. But it's also possible to restore from scratch. We have intermediary short term storage for restoring without the need of processing the source data (for fast recovery of the last few epochs) and the archived data that can be reimported and reprocessed.

### Ingestion pipelines

Ingestion pipelines are used to transport the data from the source that collects it (archiver) to the destination (elasticsearch). The data publishing part needs to be decoupled from the source to avoid problems at the source when the backend is not available. Theoretically it would be possible to send the data directly from the publisher to elasticsearch but we decided to use a message broker in between to have redundancy, temporary persistence, and the possibility to use data integration features or to scale up the processing with multiple partitions.

As a message broker we decided to use [Apache Kafka](https://kafka.apache.org/), an open-source distributed event streaming platform. One kafka cluster consists out of at least three nodes to provide proper redundancy.

A typical pipeline shipping data from the archiver to elasticsearch looks like this:

```
+--------+     +-----------+     +-------+     +----------+     +-------------+
| source | <-- | publisher | --> | topic | <-- | consumer | --> | destination |
+--------+     +-----------+     +-------+     +----------+     +-------------+
```

The publisher (aka producer) collectes the data from the source and sends messages to a kafka topic. A kafka topic is a queue of messages. A consumer reads the messages from kafka and sends the data to the destination.

Kafka allows multiple consumer groups to read from one topic and guarantees that each groups consumes each message (ordered within one partition). That allows to consume one message in multiple ways.

We store around 5 epochs of data temporarily within kafka. In case of problems we are able to replay parts of the data quickly. For redundancy we use a replication factor of `3`, that means that 2 nodes per cluster can go offline without affecting the data processing.

At the moment we have 3 ingestion pipelines running in production and one additional (events) in development.

```
+----------+           +------------------------+           +-------+           +-----------------------+
| archiver | <+------- | transactions-publisher | ------+-> | kafka | <+------- | transactions-consumer | ------+
+----------+  |        +------------------------+       |   +-------+  |        +-----------------------+       |
              |        |                        |       |              |        |                       |       |
              +------- |  tick-data-publisher   | ------+              +------- |  tick-data-consumer   | ------+
              |        +------------------------+       |              |        +-----------------------+       |
              |        |                        |       |              |        |                       |       |   +---------------+
              +------- |  computors-publisher   | ------+              +------- |  computors-consumer   | ------+-> | elasticsearch |
                       +------------------------+                               +-----------------------+           +---------------+
```

The code can be found in the following repositories: [go-data-publisher](https://github.com/qubic/go-data-publisher) and for events in [go-events-publisher](https://github.com/qubic/go-events-publisher) and in [go-events-consumer](https://github.com/qubic/go-events-consumer).


Synchronous vs asynchronous approach. Delay in both cases.

Kafka, publishers, consumers

Data storage for recovery

Transactions, tick data, computor lists, events

Cluster. Diagram.



### Query and status services

Because of the decoupling the data publishing part is asynchronous per-se. While this has some advantages it also makes the system more complex on the query side because the archive and the data collection part are not in sync but the archive is slightly delayed.

#### Status Service

To solve problems with asynchronicity and to provide metadata to the query service we decided to create a small service ([status service](https://github.com/qubic/go-data-publisher/tree/main/status-service)) that checks what data is already completely available in the data store. As a nice side effect that service also verifies the ingested data. The status service is a mediator between the query service, the archiver and the searchable data. It makes sure that only completely ingested data is served and also provides some metadata from the archiver that is available in elasticsearch.

```
+---------------+     +----------------+     +----------+
| query service | --> | status service | --> | archiver |
+---------------+     +----------------+     +----------+
  |                     |
  |                     |
  v                     |
+---------------+       |
| elasticsearch | <-----+
+---------------+
```

#### Query Service

The [query service](https://github.com/qubic/archive-query-service) replaced the most important old endpoints transparently (they return the same data but get them from elasticsearch) and created new endpoints. The new endpoints allow to specify filters and ranges and provide a more general interface for pagination. You can find the documentation for the new v2 endpoints [here](https://github.com/qubic/archive-query-service/blob/main/v2/README.md) and [here](https://qubic.github.io/integration/Partners/qubic-rpc-doc.html?urls.primaryName=Qubic%20Query%20V2%20Tree).

The new endpoints for example allow to query

The older v1 endpoints are deprecated and the ones that could not get transparently migrated to the new query infrastructure will get removed soon. Information will follow in due time.


### Modifications to the archiver

Export and import of single epochs.
Pruning of old data.
Keep compatibility (except one endpoint).

### Overall architecture

Complete diagram





# Notes (unused)



Furthermore the specialized approach only allows to access the data in certain ways and other access requires modifications that can be work and storage intensive. Additionally the archiver does not provide features out of the box that clients are used to like pagination, flexible querying and sorting. Last but not least the stored data cannot grow indefinitely and currently the complete archive including the quorum votes exceeds 500GB and grows quickly with the increased use of the network.

