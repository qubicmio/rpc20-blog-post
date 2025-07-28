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

### Ingestion pipelines

Synchronous vs asynchronous approach. Delay in both cases.

Kafka, publishers, consumers

Data storage for recovery

Transactions, tick data, computor lists, events

Cluster. Diagram.

### Databases

* Elasticsearch
* Per epoch storage. 
* Temporary storage in message broker.
* Backup and disaster recovery.

Diagram - indices

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

