+++
date = "2021-11-29T09:00:00+11:00"
title = "Tune HTTP Connection Pool for S3"
description = "Tune AWS SDK HTTP connection pool for S3 in high throughput, low latency environments"
draft = "false"
highlight = "true"
keywords = ["aws", "sdk", "s3", "performance"]
tags = ["aws", "sdk", "s3", "performance"]
+++

In this blog post, we are going to explore how to tune AWS SDK HTTP connection pool for S3 in high throughput, low latency environments. Instead of diving straight into the deep end, we will discuss why connection pooling is important and how it works. Then we will go through a series of data collected while adjusting the connection pool for optimal performance. For the purpose of experimentation, I used AWS Go SDK V2 however, the concepts discussed here are applicable to other SDKs as well.

## Background
Today, performing S3 operations in the critical path of a busy application is a commonplace. A great example is servicing a web application request using a piece of data stored in S3.

In many occasions, I have been able to get the job done with something like this.

```go
cfg, err := config.LoadDefaultConfig(context.TODO())
...
client := s3.NewFromConfig(cfg)
client.GetObject(ctx, &s3.GetObjectInput{})
```

However, in latency sensitive places, this code performs poorly. To understand why, let’s have a closer look at what’s happening here. At a high level, when invoking AWS API, number of things has to happen.

**IAM Credentials**: Calls to AWS API must be [signed](https://docs.aws.amazon.com/general/latest/gr/sigv4_signing.html) with a valid `SecretAccessKey`. Sourcing a valid `AccessKeyID` and the associated `SecretAccessKey` depends on the configuration of your application. For instance, they could be read from the environment or from EC2 metadata service (IMDS). The process of credential resolution and caching them happens when we load the configuration using `config.LoadDefaultConfig` or one of its variants. The important thing to consider here is that the credentials are cached in `Config` instance to prevent costly calls for every API request.

**Rate Limiting**: Like everything else we access over the network, AWS API calls are susceptible to intermittent failures. Good news is that SDK clients have a [built in](https://aws.github.io/aws-sdk-go-v2/docs/configuring-sdk/retries-timeouts/) retry mechanism. In Go SDK, internally, a `Retryer` is created and assigned when the service client is created via `s3.NewFromConfig(...)` or one of its variants. `Retryer` uses a token bucket implementation to limit the rate that APIs of a particular service is retried.

**HTTP Connection Pool**: Every AWS API ultimately boils down to an HTTP request to one of their service endpoints. We are going to focus on this aspect more for the remainder of this article and therefore it’s worth investing a bit of time to understand the lifecycle of an HTTP request. Go and almost all mainstream programming environments have single line library functions to invoke HTTP requests.

```Go
Http.Get("https://myservice.com")
```

When we execute the statement above, behind the scenes, it goes through a series of steps that roughly looks as follows,

- Perform a DNS query to find out the IP of myservice.com
- [Establish a TCP/IP connection](https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_establishment) with the IP endpoint resolved in previous step
- Perform [TLS handshake](https://en.wikipedia.org/wiki/Transport_Layer_Security#TLS_handshake) to establish a secure communication tunnel
- Perform HTTP [content negotiation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation) if applicable
- Send HTTP GET request and process response
- Close connection and clean-up resources

In order to minimise the overheads in these message exchanges, HTTP libraries typically employ a connection pool (more on this later). Interesting thing about AWS Go SDK is that the HTTP Client instance managing the connection pool is associated with the service client.

As we can see, service client instance holds the state (credentials cache, token bucket for retries, connection pool) vital for number of optimisations during API calls. Consequently, it’s somewhat well-known to developers that service clients should be initialised once and re-used during the lifetime of a process. It’s worth noting that service clients are safe to be shared across multiple Go routines.

## Connection Pooling
Our focus in this blog post however goes a step beyond reusing SDK clients. In particular, I want to focus on the HTTP connection pooling aspect in relation to S3. Before we dive further in, let’s take a moment to see how Go HTTP connection pool works.

![HTTP Connection Pooling in Go](/tune-http-connection-pool-for-s3/connection-pool-flow-chart.webp)
*Figure 1: HTTP Connection Pooling in Go*

Figure 1 above depicts an overly simplified version of what happens under the hood. Essentially, once we have a request ready to be sent, Go routine goes to a state waiting for a connection. If connection pooling is turned on and there’s an idle connection available, it consumes that. Otherwise, a new connection is created. After a connection is acquired in this manner, it performs the HTTP request/response and finally returns the connection to the pool if possible.

## Experiment
Assuming the connection pool is turned on, there are number of interesting questions rise out of this description. For instance, what’s the size of the connection pool? What’s the impact of under-utilising (or over utilising) the pooled connections? In order to understand this behaviour I constructed a test.

In that, I simulated a given number of concurrent Go routines performing `s3.GetObject` operation in a tight loop. Each user gets a unique s3 object to fetch to avoid any [prefix level throttling](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html) that S3 may enforce.

These tests were executed in a `c5.4xlarge` instance and each object was 1K in size. This was to ensure that the load generated by the test is not going to demand more bandwidth than what’s available.

Finally, the time it takes to fetch the object and read the body in its entirety is calculated and pushed to a influxdb instance for insights. In addition to this, I also created an [HTTP](https://pkg.go.dev/net/http/httptrace) `ClientTrace` to [measure](https://github.com/buddyspike/rabbithole/blob/main/e1/roundtripper.go) the time spent in different stages of connection life cycle.

## Results
Our first run with default settings in AWS SDK (in particular, the `MaxIdleConnsPerHost `is set to `10` by default) yielded the results in figure 2a and 2b below.

![2a 300 Go routines default connection pool settings](/tune-http-connection-pool-for-s3/2a-300-go-routines-default-connection-pool-settings.webp)
*Figure 2a 300 Go routines default connection pool settings*

![2b 300 Go routines default connection pool settings](/tune-http-connection-pool-for-s3/2b-300-go-routines-default-connection-pool-settings.webp)
*Figure 2b 300 Go routines default connection pool settings*

As we can see above, our test was making over 10K requests/sec and we used 11597 connections. What’s interesting is the conn-pool-new-vs-reused-conns graph in figure 2b, it shows us that to meet the demand we rapidly create connections. It results in the overheads we talked about earlier as seen in tls-handshake and dns-lookup graphs respectively.

Next, let’s take a look at the stats after I increased connection pool size to `128` (we will discuss more about this number below).

![3a 300 Go routines 128 MaxIdleConnsPerHost](/tune-http-connection-pool-for-s3/3a-300-go-routines-128-max-idle-conns-per-host.webp)
*Figure 3a 300 Go routines 128 MaxIdleConnsPerHost*

![3b 300 Go routines 128 MaxIdleConnsPerHost](/tune-http-connection-pool-for-s3/3b-300-go-routines-128-max-idle-conns-per-host.webp)
*Figure 3b 300 Go routines 128 MaxIdleConnsPerHost*

In this instance, we achieved higher throughput but overall nearly half as many connections as last time. If we pay close attention, we can see that conn-pool-new-vs-reused-conns graph shows a reduction in the rate that new connections are established. As a result of increased connection pooling, Go routines spend less time waiting for a new connection.

Number of connections to and from an application also impacts its CPU/memory utilisation. What’s also interesting is the CPU utilisation of the host during two tests. In the first case (figure 4), management of large number of connections pushes the limit of available CPU in `c5.4xlarge` instance.

![300 Go routines, default connection pool settings, Host CPU usage](/tune-http-connection-pool-for-s3/300-go-routines-default-connection-pool-settings-host-cpu-usage.webp)
*Figure 4 300 Go routines, default connection pool settings, Host CPU usage*

With the adjusted connection pool, we managed to lower the CPU utilisation significantly (figure 5).

![300 Go routines, MaxIdleConnsPerHost 128, Host CPU usage](/tune-http-connection-pool-for-s3/300-go-routines-max-idle-conns-per-host-128-hostcpu-usage.webp)
*Figure 5 300 Go routines, MaxIdleConnsPerHost 128, Host CPU usage*

## Points for Consideration
As we discussed, tuning of HTTP connection pool can enhance the performance of your high throughput, low latency S3 workload. However, there are number of things to consider when tweaking these knobs.

## Ideal Connection Pool Settings
An important to question to ask is, what should be the value for `MaxIdleConnsPerHost`?. Answer really depends on the number of CPUs in the host and the latency of a single operation. Number of CPUs defines how many concurrent requests can be dispatched any instance. Latency of an operations gives an idea of how many requests can be dispatched before the previously issued requests return. Finding the right multiplier requires testing your workload with different settings.

At this point, avid readers may have questioned with increased connection pool, why do we have very large number of connections. Specially shouldn’t it use just 128 connections when the test code is just 128 Go routines invoking `GetObject` in a tight loop? As it turns out, connection pooling (i.e. HTTP Keep Alive) is not purely a client side configuration. Servers can also request the client to close them. In our case, S3 servers periodically emit `Connection: close` header in its response (each time I measured this, I observed `Connection: close` header after 100 responses via a connection). I found it somewhat difficult to diagnose this behaviour without attaching a debugger because Go http package removes this header in response (Figure 6).

![S3 forcing the clients to drop connections via Connection: close header](/tune-http-connection-pool-for-s3/s3-forcing-the-clients-to-drop-connections-via-connection-close-header.webp)
*Figure 6 S3 forcing the clients to drop connections via Connection: close header*

## S3 HTTP 503 Service Unavailable
During one iteration, I observed a large number of HTTP 503 Service Unavailable responses from S3 (not 503 Slow down) resulting a spike in response times. This response is described in [AWS documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance-design-patterns.html).

>Amazon S3 automatically scales in response to sustained new request rates, dynamically optimizing performance. While Amazon S3 is internally optimizing for a new request rate, you will receive HTTP 503 request responses temporarily until the optimization completes. After Amazon S3 internally optimizes performance for the new request rate, all requests are generally served without retries.

>For latency-sensitive applications, Amazon S3 advises tracking and aggressively retrying slower operations. When you retry a request, we recommend using a new connection to Amazon S3 and performing a fresh DNS lookup.

Retry logic required is already built into AWS SDK. However, if we are aggressively pooling connections and S3 is not sending Connection: close header in the error response, it is possible that we may end-up reaching the same server instead of connecting to a new as recommended.

## DNS Lookup
One thing really peculiar about our test is DNS resolution (figure 7). At times, p99 gets significantly high. All of above tests were run with an S3 Gateway endpoint in the VPC. With a Gateway endpoint, we still use S3 public DNS name. Therefore, when the record is resolved, it would have to jump through VPC resolver to the one in S3. Another interesting fact is that S3 defines a very small TTL for DNS records to control the distribution of requests (Figure 8). As a result, in our case clients are likely to perform the forwarded DNS query more frequently. Combining these two observations, I wanted to see if there’s a way to reduce the variations in DNS lookups.

I changed my endpoint type to an interface endpoint and re-ran a `dig` to find out how it’s service endpoint it resolved. As we can see in figure 9, it has a higher TTL and the throughput(Figure 10a and 10b) was more consistent. Consider an interface endpoint if its benefits outweighs the additional cost.

![DNS lookup 300 Go routines, 128 MaxIdleConnsPerHost](/tune-http-connection-pool-for-s3/dns-lookup-300-go-routines-128-max-idle-conns-per-host.webp)
*Figure 7 DNS lookup 300 Go routines, 128 MaxIdleConnsPerHost*

![S3 DNS TTL](/tune-http-connection-pool-for-s3/s3-dns-ttl.webp)
*Figure 8 S3 DNS TTL*

![S3 Interface Endpoint DNS TTL](/tune-http-connection-pool-for-s3/s3-interface-endpoint-dns-ttl.webp)
*Figure 9 S3 Interface Endpoint DNS TTL*

![10a 300 Go routines, 128 MaxIdleConnsPerHost, S3 Interface Endpoint](/tune-http-connection-pool-for-s3/10a-300-go-routines-128-max-idle-conns-per-host-s3-interface-endpoint)
*Figure 10a 300 Go routines, 128 MaxIdleConnsPerHost, S3 Interface Endpoint*

![10b 300 Go routines, 128 MaxIdleConnsPerHost, S3 Interface Endpoint](/tune-http-connection-pool-for-s3/10b-300-go-routines-128-max-idle-conns-per-host-s3-interface-endpoint)
*Figure 10b 300 Go routines, 128 MaxIdleConnsPerHost, S3 Interface Endpoint*

## Summary
In this article we looked into what’s happening behind the scenes of an S3 `GetObject` and discussed how connection pooling can improve the situation. To summarise the key points,

- Initialise once and re-use SDK clients (this helps in many ways and not just for connection pooling)

- Connection pooling helps to minimise the overheads in DNS, TCP, TLS and HTTP.

- Default connection pool setting are not suitable for high throughput, low latency applications. Adjust the number of idle connections in the HTTP connection pool based on number of vCPUs and latency of the operation. Proximity and size are the two main contributors to latency.

- In Go, always [consume](https://github.com/buddyspike/rabbithole/blob/6e9e97c5535f7395899ffa944b687c4f86e83288/e1/client.go#L52) the `response.Body` and [call](https://github.com/buddyspike/rabbithole/blob/6e9e97c5535f7395899ffa944b687c4f86e83288/e1/client.go#L56) `Body.Close()` to ensure that connection is returned to the pool. If there’s nothing useful to do with the body, copy it to `ioutil.Discard` Writer.

- Add additional telemetry to your applications to know when networking overheads are impacting performance

Finally, the source code used in my experiments can be found in this Github [repository](https://github.com/buddyspike/rabbithole/tree/main/e1).