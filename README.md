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

One of the hardest decision was to find a suitable database for the data that needs to be served frequently and with high performance, like transactions, tick data and events. We did experiment with several solutions from relational databases to online cloud solutions but they were not able to handle the expected amount of data well. Finally we decided to go with a search engine and used [Elasticsearch](https://www.elastic.co/elasticsearch) that is based on [Apache Lucene](https://lucene.apache.org/) but offers useful features for clustering and replication.

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

To transport the data from the source that collects the data (archiver) to the destination (elasticsearch) we decided to use [Apache Kafka](https://kafka.apache.org/).

Synchronous vs asynchronous approach. Delay in both cases.

Kafka, publishers, consumers

Data storage for recovery

Transactions, tick data, computor lists, events

Cluster. Diagram.



### Query and status services

Query interface. Filters, ranges, pagination.
Handling ingestion delay.

### Modifications to the archiver

Export and import of single epochs.
Pruning of old data.
Keep compatibility (except one endpoint).

### Overall architecture

Complete diagram





# Notes (unused)



Furthermore the specialized approach only allows to access the data in certain ways and other access requires modifications that can be work and storage intensive. Additionally the archiver does not provide features out of the box that clients are used to like pagination, flexible querying and sorting. Last but not least the stored data cannot grow indefinitely and currently the complete archive including the quorum votes exceeds 500GB and grows quickly with the increased use of the network.

