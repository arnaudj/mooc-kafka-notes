Kafka series v2

# Overview

Kafka is a publish/subscribe mechanism, distributed, with fault tolerance.

# 4) Kafka Theory

## 2 Topic, partition, offsets

A topic has a name (akin to a table in a DB, without all the constraints)

A topic can be split in N partitions

Each partition has its own offset (incremental id)

Order is guaranteed per partition.

Data retention default is 1 week.

## 3 Brokers and topics

A kafka cluster has N brokers (nodes)

A broker is identified by an ID (integer).

Each partition of a topic is hosted on a dedicated broker (if possible)

Ex with 3 brokers, topic A (3 partitions), topic B (2 partitions):

```
broker1: topicA.partition3, topicB.partition1
broker2: topicA.partition1
broker3: topicA.partition2, topicB.partition2
```

## 4 Topic replication

Topic have a replication factor. It shoud be >1; ideal is 3.

Each partition of a topic is replicated on N (=replication factor) brokers.

Each partition is handled by a leader, that receives and serves the data. Replicas will sync the data (`ISR`: `in-sync replica`)

## 5 Producers and message keys

When a producer sends a message to 1 topic without any partition key: message will be load balanced accross partitions.

A producer knows which broker to connect to (to send messages) and can recover from broker failures.

Acknowledgments:

- Acks=0: producer doesn't wait or ack -> possible data loss
- Acks=1: producer waits for the leader ack -> limited data loss
- Acks=all: producer waits for leader + replicas acks -> no data loss

Message keys:

- =null: round robin
- set: set to broken hashkey%nbBrokers => same key sent to always the same partition. It can be used to enforce message ordering (ex: on an ID)

## 6 Consumers

A consumer knows which broker to connect to (to read messages) and can recover from broker failures.

Consumers reads data from a topic

Data is read in order from each partition.

### Consumer groups

Consumers read data as part of a consumer group.

Each consumer within the group reads data from exclusive partitions

If more consumers than partitions: some consumers will be inactive.

## 7 Consumer Offsets & Delivery Semantics

Consumer groups store their read offset in a kafka topic `__consumer_offsets`

Delivery semantics

- At most once (0/1): offset committed as soon as message is received -> possible data loss (if failure happens during processing)
- At least once (1..N): offset committed after processing -> preferred, but the action should be idempotent to prevent duplicates
- Exactly once: via Kafka streams (use an idempotent consumer)

## 8 Kafka broker discovery

Each broker you connect to knows about other brokers, topics and partition (metadata)

This meta data is used by the client to connect to the right broker

## 9 Zookeeper

Keeps a list of brokers.

Manages partition leader election.

Sends notification to kafka in case of change (new topic, broker up/down, etc)

Uses an odd number of servers (3,5,7), to form a quorum.

Has a leader (handles writes) and followers (handles reads)

Does not store consumer offsets anymore (since 0.1.0)

# 5) Starting Kafka

## Install

Follow https://kafka.apache.org/quickstart:

Download and unpack https://kafka.apache.org/downloads to dir

Configure the Kafka server properties in dir/config/server.properties (eg, the broker.id)

## Start

Start Zookeeper: `bin/zookeeper-server-start.sh config/zookeeper.properties`
Start the Kafka broker: `bin/kafka-server-start.sh config/server.properties`

# 6) CLI

## Configure a topic

Create a simple topic: `kafka-topics --zookeeper localhost:2181 --create --topic quickstart-events --partitions 3 --replication-factor 1`

View this topic: `kafka-topics --zookeeper localhost:2181 --topic quickstart-events --describe`

Delete a topic: `kafka-topics --zookeeper localhost:2181 --delete --topic quickstart-events`

## Producer

`kafka-console-producer --broker-list localhost:9092 --topic quickstart-events`

## Consumer

Read from the latest message (default):
`kafka-console-consumer --bootstrap-server localhost:9092 --topic quickstart-events`

Read from the latest message (default) with consumer group name
`kafka-console-consumer --bootstrap-server localhost:9092 --topic quickstart-events --group mygroup`

Read from beginning:
`kafka-console-consumer --bootstrap-server localhost:9092 --topic quickstart-events --from-beginning`

Read from offset:
`kafka-console-consumer --bootstrap-server localhost:9092 --topic quickstart-events --offset XYZ`

## Consumer groups

List groups

`kafka-consumer-groups --bootstrap-server localhost:9092 --list`

Describe group

`kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group console-consumer-76579`

Reset offsets

`kafka-consumer-groups --bootstrap-server localhost:9092 --reset-offsets --to-earliest --execute`

# 7 Kafka Java Programming 101

# 8 Kafka Real World Project

# 9 Kafka Twitter Producer & Advanced Configurations

## Acks and `min.insync.replicas`

When a producer sets `acks=all` then `min.insync.replicas` specifies the minimum number of replicas that must acknowledge a write for the write to be considered successful. If this minimum cannot be met, then the producer will raise an exception (either `NotEnoughReplicas` or `NotEnoughReplicasAfterAppend)`.

## Producer retries

Transient failures are to be handled by the caller by catching exceptions (eg, `NotEnoughReplicasException`)

Retry attempts number can be controlled by the `retries` settings, whose defaults are:

- `retries=0` for kafka <= 2.0
- `retries=MAX_INT` for kafka >= 2.1

Retry delay: `retry.backoff.ms`

Max: `delivery.timeout.ms`: An upper bound on the time to report success or failure after a call to send() returns. This limits the total time that a record will be delayed prior to sending, the time to await acknowledgement from the broker (if expected), and the time allowed for retriable send failures.

Retries can lead to out of order delivery (ex: loss of guarantee of per-partition ordering)

Fix: use idempotent producers, see below.

## Idempotent producer

Provided by kafka. Saves request IDs to do deduplication.
To set it, set property: `enable.idempotence` to `true`
Comes with these settings:

- `retries=MAX_INT`
- `max.in.flight.requests.per.connection=5` for kafka >= 1.0
- `acks=all`
