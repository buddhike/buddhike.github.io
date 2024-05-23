+++
date = "2024-05-06T09:00:00+11:00"
title = "Apache Flink Temporal Join"
description = "In depth analysis of Flink temporal join"
draft = "false"
highlight = "true"
keywords = ["streaming", "streaming-processing", "flink", "temporal-join"]
tags = ["streaming", "streaming-processing", "flink", "temporal-join"]
+++

## Background

Event time based temporal join in Flink are handy when we want to join two streams at a common point in time. In this blog post I'm going to discuss how temporal joins work in Flink (as of v1.18). This study is based on `TemporalRowTimeJoinOperator` [implementation](https://github.com/apache/flink/blob/547e4b53ebe36c39066adcf3a98123a1f7890c15/flink-table/flink-table-runtime/src/main/java/org/apache/flink/table/runtime/operators/join/temporal/TemporalRowTimeJoinOperator.java#L4).

Let's start with a hypothetical scenario where we have orders and exchange rates coming into two data streams in Kinesis. We want to emit an enriched stream where order value is converted to local currency (AUD). We begin with the schema.

```sql
%flink.ssql
DROP TABLE IF EXISTS `rates`;
CREATE TABLE `rates` (
    `currency_id` VARCHAR(3),
    `rate` DECIMAL(32,3),
    `rts` BIGINT,
    `ts` AS TO_TIMESTAMP(FROM_UNIXTIME(`rts`)),
    WATERMARK FOR `ts` AS `ts`,
    PRIMARY KEY (currency_id) NOT ENFORCED
)
WITH(
    'connector' = 'kinesis',
    'stream' = 'rates',
    'aws.region' = 'ap-southeast-2',
    'scan.stream.initpos' = 'LATEST',
    'format' = 'json'
);

DROP TABLE IF EXISTS `orders`;
CREATE TABLE `orders` (
  `order_id` BIGINT,
  `currency_id` VARCHAR(3),
  `total` DECIMAL(32,3),
  `lts` BIGINT,
  `ts` AS TO_TIMESTAMP(FROM_UNIXTIME(`lts`)),
   WATERMARK FOR `ts` AS `ts`,
   PRIMARY KEY (order_id) NOT ENFORCED
)
WITH(
  'connector' = 'kinesis',
  'stream' = 'orders',
  'aws.region' = 'ap-southeast-2',
  'scan.stream.initpos' = 'LATEST',
  'format' = 'json'
);
```

What's interesting in our schema definition is that we ask Flink to use `rates.rts` and `orders.lts` fields to generate watermarks.

> **Watermark** in Flink is current time evaluated based on event time in records flowing through the system.

Having the above tables defined, we can now create a new stream that emits orders with value in AUD using query below.

```sql
%flink.ssql
SELECT 
    o.order_id,
    o.total,
    o.ts,
    r.rate,
    r.rate * o.total AS total_aud
FROM orders o
LEFT JOIN rates FOR SYSTEM_TIME AS OF o.ts AS r ON o.currency_id = r.currency_id;
```

Here we are essentially saying that for each element in `orders` stream, find the ***latest*** currency rate in `rates` stream with a matching `currency_id` and use it to calculate `total_aud` field in the output. When choosing ***latest*** currency rate we want it to be latest relatively to order's timestamp. For instance, if we receive two exchange rate updates at 8:00am and 9:00am consecutively, we want to use the rate published at 8:00am for all orders placed between 8:00am and 8:59am. Any orders came at 9:00am or after should use the rate published at 9:00am.

## Temporal Join Implementation

If `orders` and `rates` were static data sets (e.g. tables in a relational db), we know that this operation is somewhat straightforward. In this case we have two continuous streams and number of questions may pop into our curious minds. For example, how does it scan rates efficiently? Can it operate with consistent performance as our streams grow towards infinity? What happens when data arrives out of order or delayed? I hope the following pictorial narrative of algorithm implemented in temporal join operator will help us answer those questions.

### State 0: Configuration

![Configuration](/flink-temporal-join/join-state-0.png)

Our initial state represents the operator configuration at the start of the job. In our sample scenario we have, two Kafka topics (orders and rates), corresponding Kafka source operators read data out of those topics and temporal join operator receive data from source operators for joining.

Each box in the diagram also depicts important state maintained in operators and the watermark either generated or seen over time.

### State 1: Data flow in left
![Data flow in left](/flink-temporal-join/join-state-1.png)

On the left side of our join we have orders topic. As data flows through Kafka source operator, temporal join operator adds orders to a hash map where key is a monotonically increasing number and value is order itself. Join operator uses keys in the map to ensure that it's output is emitted in same order as input.
In addition to storing incoming orders in the map, join operator sets a timer to be triggered at the event time if event time of order is the earliest observed thus far.

### State 2: Data flow in right
![Data flow in right](/flink-temporal-join/join-state-2.png)

On the right hand side of our join we have rates topic. As data flows through corresponding Kafka source operator, temporal join operator adds them to a hash map where key is event time of rate and the value is the rate itself. Similar to left side data flow handling, join operator also sets a timer to be triggered at the event time if event time of rate is the earliest observed. 

In both left and right data flow cases, existing timer is overwritten leaving only one active timer at any given point.


### State 3: Watermarks are generated
![Watermarks are generated](/flink-temporal-join/join-state-3.png)

Eventually, both source operators will generate corresponding watermarks which in our case is the latest event time it has observed thus far. Temporal join operator considers the smaller of two watermarks as the current watermark. Watermark signifies the progress of time. When time progress, Flink triggers any timers due. In our example scenario, timer we set for time `1` is triggered at this point and it performs the following logic to join data stored in left and right maps.

- Take all items from right hand side where event time is less than or equal to current watermark 
  - Set a new timer to be triggered for the earliest event time in remaining records
- For each item find the corresponding record on the right hand side based on event time
  - Emit the resulting record to downstream operators
- Remove the items retrieved in step 1 from left hand side
- Go through right hand side and remove all items with older event time than watermark except latest

## Key Points to Consider 

It's now time to discuss a few important facts about this implementation.

- Flink timers are triggered only when time is progressed. In our example above, orders received with timestamp `1` after watermark is progressed to `1` will accumulate in join operator's state. They are cleared either when time is progressed or if you have configured `table.exec.state.ttl` configuration parameter. When `table.exec.state.ttl` parameter is set, join operators state is cleared if it stays idle for the duration specified.

- Watermark at join operator is the minimum of two upstream operators. This has an interesting consequence if one side of our join does not produce data as fast as the other side. As a matter of fact, this rule applies to all joins in Flink. Interesting thing is, many use cases using temporal join for enrichment workflows like above deal with a slowly changing dimension. We can mitigate this scenario by setting `table.exec.source.idle-timeout` in config. When `table.exec.source.idle-timeout` is set, idle source's watermark is temporary disregarded when progressing watermark.

```sql
%flink.ssql
SET 'table.exec.source.idle-timeout' = '5s';
SELECT 
    o.order_id,
    o.total,
    o.ts,
    r.rate,
    r.rate * o.total AS total_aud
FROM orders o
LEFT JOIN rates FOR SYSTEM_TIME AS OF o.ts AS r ON o.currency_id = r.currency_id;
```

- Contrary to other joins in Flink, temporal join operator does not drop late arriving records. For inner join scenario, as long as there's a matching version on the right side (because we had late arriving records for both left and right), we get a result. For left join scenario, we get a result with a `NULL` value for unmatched version.

- Not all enrichment use cases can fit into the logic implemented by temporal join. For instance, there might be an instance where we need to match elements on left to the closest version on the right. For this type of requirements, we could consider implementing our logic in [`CoProcessFunction`](https://nightlies.apache.org/flink/flink-docs-master/api/java/org/apache/flink/streaming/api/functions/co/CoProcessFunction.html)
