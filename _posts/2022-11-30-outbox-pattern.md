---
layout: post
title: Outbox pattern
categories: [patterns]
tags: [event-driven-architecture,architecture,dual-write,kafka,eventual-consistency,outbox,outbox-pattern,outbox-table-pattern,two-phase-commit]
mermaid: true
---

# Outbox Pattern

## Problem

Let's picture a common scenario where we have a service that manages `Products` and our customer wants to change a property of a given product, let's say its `price`. Several other systems might be interested in the price change information, for instance the *marketing* team could have some triggers configured to send e-mails to certain customers about offers.

To solve that we can use `Kafka` as event bus, so when the price is updated by the *product-service*, it can publish a new event to a kafka topic and then the *marketing-service* can consume it and have the new value for the price of that product.

```mermaid
    graph LR

    database[( product-database)]
    kafka("Kafka[price-change-topic]")

    User -- changes price --> product-service 
    product-service -- updates price --> database    
    product-service -- publishes new price --> kafka
    kafka -- consumes price changes --> marketing-service

```

Ok, but the **product-database** and **price-change-topic** are different external systems from the **product-service**, they are separated by a *network* connection and we have one state that we need to propagate to them and in most of the cases we want to ensure that they agree to with each other, even if not immediately.

``Writing to both using database transaction`` is the first thought that might come to mind, but that can be misleading. There is not shared transaction boundary between database and kafka topic but rather there are two. Our product service would need to synchronize between these two systems and perform some form of two-phase commit to keep systems in sync. Lets see scenarios that can happen:

|  | database ✅ | database ❌ |
| --- | --- | --- |
|__kafka ✅__| happy path | send rollback event |
|__kafka ❌__| rollback in DB | noop |

 In a  situation where the transaction is rolled back after the communication with Kafka is flushed, you would end up with a *ghost* event on the topic that would need to be handled by sending rollback event. Depending on the publishers configuration it could also happen that the event would stay on the buffer to be send even after the transaction is committed, and in a case of an interruption on the system those events would be lost.

The problem is in delivery guarantees of the data. There are couple options that can happen:

* at most once delivery - best effort but no guarantees for durability. In edge cases some data may be lost. This is common pattern for storing logs.
* exactly once delivery - the data is not duplicated and producer guarantees full synchronization between kafka and database. This is very hard to achieve.
* at least once delivery - the data may be duplicated but producer guarantees delivery at least once. Usually events are not duplicated but during edge case scenario it can happen (f.e. network partition). This is usual compromise to go, and consumers of data need to compensate. Usually consumers apply deduplication window based on their use-case.

How can we ensure that the state between the two storage systems is consistent? Meaning, when I have the price stored in the database I'll have it on the topic as well.

## Implementing the Outbox Pattern

The Outbox Pattern is a trick where we use the database transaction to ensure no events are lost guaranteeing at least once delivery.
You can simply implement it by adding a second transaction. First, storing the event to the database with a `pending` state and once it is published you update it to `published`. In the case of something goes wrong events can be retried, and the worst inconsistency that can happen is having duplicated events on the topic. Let's see all the scenarios bellow.

## Happy Path

```mermaid
    sequenceDiagram
        actor User
        User->>product-service: calls post endpoint
        product-service->>+product-database: store the price change
        product-service->>product-database: store the event with status `pending`
        product-database->>-product-service: committed
        product-service->>kafka:publish
        kafka->>product-service: ok
        product-service->>+product-database: update the event with status `completed`
        product-database->>-product-service: committed
        product-service->>User: success response

```

---

## Failure to publish to Kafka

When there is a failure to publish to kafka, the service can retry from the database and make sure that no messages are lost. Furthermore if you want to ensure that the order of the events are preserved, you can go for publishing the events always through a secondary process that would select from the database and publish to kafka.

The downside of this approach is the eventual increase on the sla of the delivery of the event. That's the trade off to gain the eventual consistency.

```mermaid
    sequenceDiagram
        actor User
        User->>product-service: calls post endpoint
        product-service->>+product-database: store the price change
        product-service->>product-database: store the event with status `pending`
        product-database->>-product-service: commit
        product-service->>User: success response
        loop
            product-service->>+product-database: find pending events
            product-service->>kafka:publish
            kafka-->>product-service: |network exception|
            product-service->>kafka:publish
            kafka->>product-service: ok 
            product-service->>+product-database: update the event with status `completed`
            product-database->>-product-service: committed
        end
```

tip: If you have multiple nodes of the product-service, you still need to take care of the order of the events.

* One solution is to use a leader election and do the publish on a single node, using the Single Producer Principle. Although if you are in a high throughput that can be a bit problematic and slow your business down.
* Another solution is to attribute a specific partition or group of partitions to a specific node, than you can select from the database using the partition key and that would still keep the order across the multiple nodes. Down side is a bit of configuration and you would still lock your nodes to the size of partitions and that would make scaling harder.

---

## Change Data Capture

Change Data Capture connects directly to your database, reading its log files, it infers the changes from the logs and publish the events to Kafka directly. If any error occur on the publish, it can take cares of the retry for you, and it is all transparent.
One popular implementation of CDC is the [Debezium](https://debezium.io/) project.

```mermaid
    graph LR

    database[(product-database)]
    kafka("Kafka[product-topic]")
    User -- changes price --> product-service 
    product-service -- updates price --> database    
    Debezium -- publishes new product version --> kafka
    kafka -- consumes product version --> marketing-service
    database --- Debezium

```

```mermaid
    sequenceDiagram
        actor User
        User->>product-service: calls post endpoint
        product-service->>+product-database: store the price change
        product-database->>-product-service: commit
        product-database->>Debezium: observes the change
        Debezium->>kafka:publish
        product-service->>User: success response
```

---

### Outbox Table Pattern

Now let's put all together!

CDC is great for `Event Sourcing` when we publish the the state of the event every time, which is great to replicate data across systems, like in the case where we wanted to propagate the price. But what if we have an use case of `Event Streaming` where we want to expose a fact that happened, for instance `product-sold-event`. In this scenario we would not have exactly an entity that we want to propagate but a new event that we want to publish. Still that can follow the same pattern as the CDC. You just need to create a table that would match your new event and store them there instead of publishing them directly. Eventually you would need to clean up this table to not let it grow forever.

This approach can be very powerful, because it can also allow you to de-normalize data that from different sources and then provide it as events downstream, giving this data a new purpose.  
Let's consider this simple example below: (ignore the super simple and non realistic modeling)

```mermaid
    erDiagram
    
    Product {
        string SKU PK "Identifier of the product"
        string name
        double price
    }
    Stock ||--|| Product : ""
    Stock {
        string id PK "stock id"
        string productId FK "id of the product"
        int quantity
    }

```

Depending on your domain, you might have multiple bounded contexts that would benefit from having an event that would already group the both entities, `Product` end `Stock`. So, from the `product-service` domain can receive two distinct events each of the entities, then we can join then and further use the CDC to publish it downstream.
Also to note that I'm picturing here `Product` end `Stock` as two entities in my database, but they could also be events from different systems.

```mermaid
        graph LR

        database[(product-database)] --> Debezium
        Debezium --> kafkaProduct("Kafka[product-topic]")
        Debezium --> kafkaStock("Kafka[stock-topic]")
        kafkaProduct -->  composer-service
        kafkaStock -->  composer-service
        composer-service -- Joins to database --> composerdb[(composer-database)] 
        composerdb --- Debezium2
        Debezium2 --> kafkaProduct2("Kafka[productEnhanced-topic]")


```

```mermaid
    erDiagram
  
    ProductStock {
        string name
        double price
        string stockId 
        string productId 
        int quantity
    }

```

## NoSql no problem

Oh you are not using a sql database and still want to benefit from this pattern? [Confluent](https://www.confluent.io/) provides a long list of connectors that allows you to connect your storage to kafka, making easy the data replication. The list goes from `Elasticsearch`, `Kinesis`, `MongoDb` to many more.

## Summary

Data replication across the organization and sometimes even beyond the organization is a challenge and an opportunity that we face everyday. By using these simple tricks we can achieve the right consistency without trading off the latency too much. Of course if absolute latency is your concern, and you are on the *p999 < 100 milliseconds* SLAs maybe you should consider other communication patterns.
