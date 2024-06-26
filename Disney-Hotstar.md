# How Disney+ Hotstar delivers 5 billion emojis in real-time ?

**Answer:** 

- They send emojis to the server over HTTP and run the API server in the Go programming language

- They don’t do caching because emojis must be shown in real-time

- They avoided blocking resources and supported high concurrency via asynchronous processing

- They buffer the data and write it asynchronously in batches to the message queue (Kafka)

- They use an in-house data platform to run Kafka due to its high operational complexity

- They use Spark to process the stream of emojis from Kafka

- They write the aggregated emojis into another Kafka

- They sent the aggregated emojis to the PubSub infrastructure

- They use the MQTT protocol on PubSub to deliver the emojis to the client

- They use EMQX message broker to distribute the messages

- They use the reverse bridge architecture to scale the EMQX cluster.



## Brief Analysis and Explanation :

### Hotstar Architecture
- They don’t do caching because emojis must be shown in real-time.

- Also they split the services into smaller components to scale independently.
Besides they avoided blocking resources and supported high concurrency via asynchronous processing.

![Hotstar Architecture](https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F513e7007-2464-4393-9313-de772f9991e9_1274x469.png)


**Here’s how they scaled emojis to millions of concurrent users:**

**1. Receiving Emojis From Clients**
----
They send emojis from the `client to the server using HTTP`. And run the API server in the `Go` programming language.

The data gets written to a local buffer before returning success to the client. Because it prevents hogging client connections on the API server.

And the buffered data is asynchronously written in batches to the message queue. They use `Goroutines` while writing to the message queue for concurrency.

`Goroutines are lightweight threads that execute functions asynchronously.`

![Asynchronous Processing on API Server](https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fbfcf1d9c-2472-4256-ab49-c3dbd217bcaf_1286x392.png)


They use `Apache Kafka` as the message queue. It decouples system components and automatically removes the older messages on expiry.

Yet Kafka has a high operational complexity. So they use their in-house data platform, `Knol to run Kafka.`

The message queue is a common mechanism for asynchronous communication between services.

**2. Processing Emojis**
----

They use [Apache Spark](https://spark.apache.org/) to process the stream of emojis from Kafka.

Stream processing is a type of data processing designed for infinite data sets.

A streaming job in Spark aggregates the emojis over smaller intervals to offer a real-time experience. And the aggregated emojis get written into another Kafka.

![Normalizing Stream of Emojis](https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fc74a88b8-344b-411e-860b-a312fefdee0e_1052x293.png)


They chose Spark because it offers micro-batching and good community support. Also it offers fault tolerance through checkpointing.

`Micro-batching is batching together data records every few seconds before processing them.`

`While Checkpointing is a mechanism to store data and metadata on the file system.`

Besides combining Spark and Kafka guarantees only one-time processing of emojis. Thus preventing duplicates.


**3. Delivering Emojis to Clients**
----

They use `Python consumers` to read normalized emojis from Kafka.

And the emojis get sent to the `PubSub` infrastructure.

While the PubSub delivers emojis over a persistent connection to the clients.

They use the `Message Queuing Telemetry Transport (MQTT)` protocol to deliver the emojis.

`MQTT is a lightweight messaging protocol over TCP. And it’s supported by a broad range of platforms and devices.`

![PubSub Delivering Emojis](https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fb1f3d1e3-7179-4617-8eb1-94a55d53a6f3_1302x284.png)


They use the `EMQX (MQTT)` message broker to distribute messages. It’s based on `Erlang` and offers very high performance.

`A single machine in the EMQX cluster could handle 250k connections.`

But `EMQX internally uses a distributed database called Mnesia.` And it was designed to support only a few machines. So it became difficult to scale beyond 2 million concurrent connections by adding more machines.

Thus they didn’t use the EMQX cluster. Instead set up a multi-cluster system using the reverse bridge.

![Scaling EMQX Cluster Using Reverse Bridge](https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F499b9414-d330-406b-8ff5-0d5225579cb3_2441x1013.png)


Each cluster contains a single publish node and many subscribe nodes.

While the main publish node forwards the emojis to each cluster.

They run a `Golang service` on each cluster. And it subscribes to all the messages from the main publish node and forwards them to the publish node on every cluster. They called it the reverse bridge architecture.

Besides they set up autoscaling to change the number of subscriber nodes for scalability.

----