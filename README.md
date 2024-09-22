## What we need

- building aggregator service
- consume from 7 services via message broker (kafka)
- aggregation data 

## Problem

- handling high traffic
- handling message in order
- handling deduplicate message

## Solution
### Integration with Event-Carried State Transfer

- Consumers are much less likely to need to make a request back to the producer for more information when they communicate with event-carried state transfer. State transfer is great for interested consumers to build a local representation of the data so that it may handle future requests independently.

Some uses for event-carried state transfer are presented here:

- Storing shipping addresses for customers in a warehouse service.
- Building a history of product purchases for a seller in an independent search component.
- Information from multiple producers can be combined to create entirely new resources for the application to support additional functionality.

The primary advantage of event-carried state transfer is that consumers are temporally decoupled from producers. The availability of the producer is no longer a factor in how resilient the consumer will be when handling requests that it receives.

### Eventually Consistency

- An eventually consistent system that has stopped receiving modifications to an item will eventually return the same last update across the system.
- In distributed applications and especially in event-driven applications. It is a trade-off made for the performance and resiliency gains when choosing to architect a system with asynchronous communication patterns.
![Eventually System](./images/eventually-system.png)

### Message Delivery Guarantees

- **At-most-once message delivery**
  
  The producer does not wait for an acknowledgment from the message broker when it publishes a message under the at-most-once delivery model, as depicted in the following diagram:
![at-most-once](./images/at-most-once.png)
  Message deduplication and idempotency are not concerns. However, the possibility that the message never arrives is very real. In addition to the producer not confirming that the message broker received the message, the broker does not wait for any acknowledgment from the consumer before it deletes the message. If the consumer fails to process the message, then the message will be lost.
- **At-least-once message delivery**
  
  With at-least-once delivery, the producer is guaranteed to have published the message to the message broker, and then the broker will keep delivering the message to the consumer until the message broker has received an acknowledgment that the message has been received, as depicted in the following diagram:
![at-least-once](./images/at-least-once.png)
  A consumer can receive the message more than once, and they must be utilizing either message deduplication or have implemented other idempotency measures to ensure that the redelivery of a message does not result in it being processed more than once.
The reasons why a message might be delivered more than once can vary, but it will often be because the message broker is waiting a limited amount of time for an acknowledgment from the consumer. If the consumer takes too long to send an acknowledgment, then the message is re-queued to be sent again. Systems that can deduplicate messages so that repeated deliveries only result in one processing instance are the ideal use case for at-least-once delivery.
- **Exactly-once message delivery**
  
  Having a guarantee that a message will arrive exactly once is not so simple. As with the at-least-once delivery guarantee, the producer will wait for an acknowledgment from the broker. Also, the broker will keep delivering the message until it has received an acknowledgment from the receiver, as depicted in the following diagram:
![exactly-once](./images/exactly-once.png)
  What is different now is that what received the message was not the consumer but instead an additional component that holds a copy of the message. The message can then be read, processed, and deleted by the consumer. That is at least the idea of how exactly-once delivery can be achieved, but network reliability and issues with the message broker or with the message store can all still cause the process to fail.

**Note:** Exactly-once delivery would be ideal for just about any situation, but it is extremely hard or downright impossible to achieve in most cases.

### Message Deduplication

- **Producer Message Deduplication (Idempotent Producer)**

  Producer idempotence ensures that duplicates are not introduced due to unexpected retries.
  ![idempotent-producer](./images/idempotent-producer.png)
  When enabling producer idempotence, each producer gets assigned a Producer ID (PID) and the PIDis included every time a producer sends messages to a broker. Additionally, each message gets a monotonically increasing sequence number (different from the offset - used only for protocol purposes). A separate sequence is maintained for each topic partition that a producer sends messages to. On the broker side, on a per partition basis, it keeps track of the largest PID-Sequence Number combination that is successfully written. When a lower sequence number is received, it is discarded.

  ``Starting with Kafka 3.0, producer are by default having enable.idempotence=true and acks=all.``
- **Consumer Message Deduplication (Idempotent Consumer)**
  
  _**Manual Offset Commit**_
  If an application fails to process a message due to a crash or any other reason like one of the dependencies is not met. The message will be lost since the message offset is already committed by the consumer automatically.
  ![duplicate-message-fail-auto-commit](./images/duplicate-message-fail-commit.png)
  Another problem could lead to duplicate messages. Application processed the message but it was unable to commit the offset due to consumer crash or Kafka unavailability or some other reasons.
  ![duplicate-message-fail-process](./images/duplicate-message-fail-process.png)
  
  By handling the offset commit manually and only committing the message offset once the application finishes its processing using synchronous API Commit, At-Least-Once delivery semantic can be guaranteed since the offset corresponding to a message is only committed after the message has been successfully processed
  
  _**Message Inbox Management**_
  Manual message management has many complexities like persisting message id at application side for duplication reference, having separate table and schema for message id management, etc. However, this approach of transactional inbox pattern can ensure the message will be processed once only in the consumer.
  ![message-inbox](./images/message-inbox.png)

### Message Consumption Ordering
- Kafka only provides a total order over messages within a partition, not between different partitions in a topic. Per-partition ordering combined with the ability to partition data by key is sufficient for most applications. However, if a total order over messages is required, this can be achieved with a topic that has only one partition, though this will mean only one consumer process.
- A Kafka producer sends messages to a topic, and messages are distributed to partitions according to a mechanism such as key hashing. 
  In case the key (key=null) is not specified by the producer, messages are distributed evenly across partitions in a topic. This means messages are sent in a round-robin fashion (partition p0 then p1 then p2, etc... then back to p0 and so on...).
  If a key is sent (key != null), then all messages that share the same key will always be sent and stored in the same Kafka partition. A key can be anything to identify a message - a string, numeric value, binary value, etc.
  Leveraging such feature of Kafka can ensure the message will be sent to a specific topic partition and thus, maintaining the original nature of message ordering for each partition.
  ![message-order](./images/message-order.png)