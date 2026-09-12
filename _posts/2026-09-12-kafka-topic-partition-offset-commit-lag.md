---
title: Apache Kafka - Topics, Partitions, Offsets & the Commit Log
categories: [Kafka, Streaming]
comments: false
tags: [apache-kafka, event-streaming, distributed-systems]
date: 2026-09-12 00:00:00 +05:30
---

Ahoy, curious engineers! ☕

In our [previous Kafka blog](https://kraghu306.github.io/posts/introduction-concepts-of-kafka/), we built the basic mental model of Apache Kafka - what Kafka is, why it is different from traditional messaging systems, and where it fits in real-world architectures.

Now that we know **why Kafka exists**, it is time to understand **how Kafka actually stores and organizes events**.

This is where a few important Kafka concepts come together:

- Topics
- Partitions
- Offsets
- The commit log

And once these pieces are clear, Kafka starts to feel much less mysterious.

Let's dive in.

### A Quick Recap

We previously looked at Kafka as something like:

`producer` -> `Kafka` -> `consumer`

But this picture hides an important question:

**Where exactly does Kafka put the data?**

If millions of events are continuously flowing into Kafka, Kafka needs a way to organize them, store them, and allow consumers to read them efficiently.

This is where **topics and partitions** come into the picture.

### What is a Kafka Topic?

A **topic** is a logical name for a stream of events.

Imagine an e-commerce application.

We might have events such as:

`OrderCreated`

`OrderPaid`

`OrderShipped`

`OrderDelivered`

Instead of mixing everything together, we can create separate topics:

`orders`

`payments`

`shipments`

A producer writes events to a topic, and consumers read events from that topic.

So, at a very high level:

`producer` -> `topic` -> `consumer`

One important thing to remember:

> A topic is a logical concept. The actual data is stored inside partitions.

And this is where Kafka becomes interesting.

### Why Do We Need Partitions?

Imagine our `orders` topic receives **10 million events per second**.

If all of those events had to be written to a single place, that single place would eventually become a bottleneck.

Kafka solves this by splitting a topic into multiple **partitions**.

For example:

```text
orders

Partition 0
Partition 1
Partition 2
Partition 3
```

Now the workload can be distributed.

Instead of thinking:

`10 million events` -> `one place`

we can think:

`10 million events` -> `multiple partitions`

Each partition is an ordered sequence of events.

This gives Kafka one of its most important properties:

**horizontal scalability.**

We can add more partitions and distribute the workload across multiple Kafka brokers.

### A Partition is an Ordered Log

Let's zoom into one partition.

Suppose our `orders` topic has a partition containing:

```text
OrderCreated
OrderCreated
OrderCreated
OrderCreated
OrderCreated
```

Kafka does not randomly arrange these events.

They are appended to the partition in order.

Conceptually:

```text
+---------+---------+---------+---------+
| Event 0 | Event 1 | Event 2 | Event 3 |
+---------+---------+---------+---------+
     |         |         |         |
     v         v         v         v
  Order A   Order B   Order C   Order D
```

New events are appended to the end.

`Event 0` -> `Event 1` -> `Event 2` -> `Event 3` -> `Event 4`

This append-only design is a major reason Kafka can handle high write throughput efficiently.

But now another question appears:

**How does a consumer know which event it should read?**

Enter the **offset**.

### What is an Offset?

An **offset** is the position of an event within a partition.

For example:

```text
Partition 0

Offset:   0        1        2        3        4

Event:    A        B        C        D        E
```

If a consumer has processed event `C`, it knows where it is in the partition.

The offset gives Kafka consumers a way to answer:

> "Where am I in this stream?"

This is a very important difference from many traditional messaging systems.

Kafka does not need to delete an event simply because one consumer has read it.

The event remains in Kafka according to the topic's retention configuration.

This means another consumer can potentially read the same event later.

### Why Does Kafka Keep the Events?

This is one of the concepts that makes Kafka different from a simple queue.

Imagine three applications:

```text
orders topic
      |
      +----> Payment Service
      |
      +----> Analytics Service
      |
      +----> Notification Service
```

The payment service may care about payment processing.

The analytics service may care about business metrics.

The notification service may care about sending updates to customers.

All three services can independently consume the same stream of events.

Kafka does not say:

> "The Payment Service consumed this event, so nobody else can see it."

Instead, Kafka keeps the event and lets consumers maintain their own position.

That position is represented by the consumer's **offset**.

### The Commit Log

Now we can introduce another important term:

**Commit log.**

A Kafka partition can be thought of as an append-only log.

Something like:

`0` -> `1` -> `2` -> `3` -> `4` -> `5` -> `6`

Each number represents an offset, and each position represents an event.

New events are appended to the end:

`0` -> `1` -> `2` -> `3` -> `4` -> `5` -> `6` -> `7` -> `8`

Kafka does not normally modify an event sitting in the middle of this sequence.

This gives us an important mental model:

> Kafka is fundamentally an append-oriented event log.

This is why concepts such as **ordering, offsets, replay, and retention** are so important when working with Kafka.

### Ordering in Kafka

There is an important detail here.

Kafka guarantees ordering **within a partition**.

For example:

```text
Partition 0:

Event A -> Event B -> Event C -> Event D
```

A consumer reading that partition sees those events in that order.

But imagine:

```text
Partition 0: A -> B -> C

Partition 1: X -> Y -> Z
```

Kafka does not provide one global ordering across both partitions.

This distinction becomes extremely important when designing Kafka-based systems.

If the order of events matters, we need to think carefully about **which partition an event should go to**.

### How Does Kafka Decide the Partition?

When a producer sends an event to a topic, Kafka needs to determine which partition should receive it.

A common approach is to provide a **key**.

For example:

```text
Topic: orders

Key: customer-101
Value: OrderCreated
```

The producer can use the key to determine the partition.

This can be useful because events for the same key can be routed to the same partition.

For example:

```text
customer-101 -> Partition 2
customer-101 -> Partition 2
customer-101 -> Partition 2
```

Now events for that customer can maintain their ordering within that partition.

This is a powerful concept, but it also means partitioning is not just a performance decision.

It can be a **data-modeling decision**.

### A Simple Mental Model

At this point, let's connect the concepts we have learned.

Think of a Kafka topic as a collection of ordered logs:

```text
Topic: orders

Partition 0
0 -> 1 -> 2 -> 3 -> 4

Partition 1
0 -> 1 -> 2 -> 3

Partition 2
0 -> 1 -> 2 -> 3 -> 4 -> 5
```

Each partition has its own sequence of offsets.

So when we say:

`Offset = 3`

we also need to know:

`Which partition?`

An offset by itself is not globally unique within a topic.

The combination of:

`Topic + Partition + Offset`

gives us the position of an event in Kafka.

This is a small detail, but an important one to keep in mind.

### Replay - One of Kafka's Superpowers

Now let's connect everything together.

Suppose a consumer processes:

`Offset 0`

`Offset 1`

`Offset 2`

`Offset 3`

`Offset 4`

And later we discover that there was a bug in our application while processing events.

Because Kafka retains the events, we can potentially move the consumer's position back and process the events again.

This is called **replaying the stream**.

Conceptually:

```text
Normal processing:

0 -> 1 -> 2 -> 3 -> 4 -> 5

Replay:

0 -> 1 -> 2
          |
          +-- start again
```

This is one of the biggest mental shifts when learning Kafka.

The event is not simply:

> "A message that was delivered."

It is:

> "A record of something that happened and is retained for consumers to process."

That is why Kafka works so well for event-driven architectures.

### Retention - Does Kafka Store Events Forever?

Not necessarily.

Kafka topics have retention configurations.

For example, a topic might retain events for:

`7 days`

After the retention period, Kafka can remove older data.

The important point is:

**consumption and deletion are separate concepts.**

A consumer reading an event does not automatically mean:

`read` -> `delete`

Instead:

`produce` -> `store` -> `consume` -> `retain` -> `eventually expire`

This separation is what makes replay and multiple independent consumers possible.

### Putting It All Together

Let's now connect our concepts.

Imagine an application producing order events:

```text
                Producer
                   |
                   v
             orders topic
                   |
          +--------+--------+
          |        |        |
          v        v        v
       Part 0   Part 1   Part 2
          |        |        |
          v        v        v
       offsets  offsets  offsets
```

The flow is:

1. A producer creates an event.
2. The event is written to a topic.
3. Kafka places the event into a partition.
4. The event receives an offset within that partition.
5. Consumers read events from the partitions.
6. Consumers track their progress using offsets.
7. Events remain available according to the topic's retention policy.
8. Consumers can potentially replay events when required.

This is the basic storage model of Kafka.

Once this model becomes clear, many advanced Kafka concepts become much easier to understand.

### Let's Get Kafka Running

Understanding diagrams is useful.

But Kafka becomes much easier once we actually see events flowing.

For the next step, we can run Kafka locally and create our first topic.

The basic journey will look something like:

```text
Start Kafka
    |
    v
Create Topic
    |
    v
Start Producer
    |
    v
Send Events
    |
    v
Start Consumer
    |
    v
Read Events
    |
    v
Inspect Partitions & Offsets
```

Once we do this, the concepts we discussed today will stop being abstract.

We will actually see:

`Topic` -> `Partition` -> `Offset` -> `Consumer`

working together.

### What We Learned Today

Let's pause and lock in the important concepts.

- **Topic** - a logical stream/category of events.
- **Partition** - an ordered sequence of events within a topic.
- **Offset** - the position of an event inside a partition.
- **Commit log** - the append-oriented log model used to store events.
- **Retention** - controls how long Kafka keeps events.
- **Replay** - the ability to read retained events again from an earlier position.
- **Ordering** - guaranteed within a partition, not globally across all partitions.

If these concepts are clear, we now have a solid foundation for understanding Kafka's architecture.

### What's Next?

In the next blog, we'll stop talking about Kafka only through diagrams and **run it locally**.

We'll create our first Kafka topic, produce events, consume them, inspect partitions and offsets, and see what actually happens when events start flowing through Kafka.

We'll also start looking at an important question:

**Where does Kafka actually store all these events?**

That's where the physical side of Kafka starts becoming interesting - log segments, broker storage, replication, and fault tolerance.

The journey is getting interesting.

Thanks for joining this second leg of our Kafka expedition.

Remember: a Kafka topic is not just a place where messages wait to be consumed. Think of it as a **stream of facts, organized into partitions, with every event having a position in that stream**.

Until next time, stay curious and keep building. 🚀
