+++
date = "2024-02-25T09:00:00+11:00"
title = "Journey of libvegas (Part 1) - Inception"
description = "Journey of libvegas (Part 1) - Inception Building a new Kinesis Client Library"
draft = "false"
highlight = "true"
keywords = ["aws", "kinesis", "client", "streaming", "streaming-analytics", "go"]
tags = ["aws", "kinesis", "client", "streaming", "streaming-analytics", "go"]
+++

A few years ago, I wanted to dabble with an AWS service that I hadn't used before. One day, I tried Amazon Kinesis Data Streams (KDS). I was shocked
by how little has been said about it. It's a stream storage service that
is completely serverless, can auto scale (both in and out without moving data around) and it's designed for applications with high throughout and low latency in mind 🤯.

Having worked in large development teams before, another thing that I was 
impressed by was the fact that with KDS you can design and plan your workloads
without considering your neighbours. If you want to think of data streams as a piece of first class application infrastructure that your developers can use, KDS is a compelling technology.

Recently I started working on [libvegas](https://github.com/buddhike/libvegas), a new KDS client library to simplify client side programming model. It's very early days for this project and I plan to share my design notes here.

## Goals
We set off for this journey with some broad set of goals. As we make progress, we will refine this list. 

### Simple programming model
Using KDS with `libvegas` should be effortless. API is designed around the two
tasks that developers wish to deal with - send and receive, nothing more, nor less.  

To send data using Python client, you instantiate an instance of `Producer` 
class and invoke `send` method on it.

```python
import vegas
p = vegas.Producer("stream-name")
p.send("partition-key", b"message")
```

Similarly, on the receiving side, instantiate an instance of `Consumer` class
with a callback function that gets automatically invoked upon the arrival of
new records.

```python
import vegas 

def process_record(r):
    print(str(r.partitionKey) + ":" + r.data.decode("utf-8"))

c = vegas.Consumer("stream-name", "efo-arn", process_record)

input("Consumer is running. Press ctrl+c to stop\n")
```

For now, let's just say that Producer and Consumer API should hide the details of complexities in sending and receiving data using a simple interface like the one above. As an example, it should guarantee the order of messages even if KDS backend configuration changes while the records are in transit. We will discuss various implementation options to tackle each one of these challenges in separate blogs.

### Consistent programming model in mainstream programming languages
Developers should be able to enjoy the succint producer/consumer API in their favourite programming language. Currently we have `Python`, `JavaScript`, `Java`, `C#`, `C`, `C++`, `Ruby` and `Go` in our list.

We want the developer experience of using KDS to be consistent across all programming languages. By consistency here we mean two things. Firstly, we want `Producer` and `Consumer` API to be same across supported programming languages. Secondly, we want developers to be able to use the toolchain they are familiar with. For instance, a `nodejs` developer using `npm` package manager should be able to install the library via `npm install libvegas` in their development environment and expect the package to work without additional non `nodejs` dependencies.

### Send batches while preserving the order 
Today, KDS `PutRecords` API can be used to send batches of records to maximise available network bandwidth. One challenge with this API is that it does not guarantee the order of records in the batch. In `libvegas` we want to ensure that order of records is preserved whether batching is used or not.

### Extensible
Much of the features required for stream processing can be packed into client
libraries. We want `libvegas` to be extensible so that the features such as 
message de-duplication and windowed aggregations can be built once and used anywhere.

### Observability
We want to provide visibility to the machinery used to send and receive data.
We will emit useful metrics such as messages sent/received/sec using an open 
interface so that they can piped to a monitoring tool of choice.
Detailed logs are also written via a well-defined interface so that they also can be captured using the logging framework used to develop the client application.

### Auto scaling consumers
We should be able to add or remove consumers to process data in the stream based on the workload. As the consumer fleet capacity changes, library should strive to achieve a balanced distribution of work across the available compute.

Each one of these goals required much in depth discussion however, in this 
introductory blog post, I wanted to share a glimpse of what we are about to embark on and receive early feedback. 

In the next post we will discuss how we are planning to develop `libvegas` in 
`Go`.
