
---
title: "Go & Apache Kafka"
draft: false
date: "2021-02-23"
---


## Kafka Go Client Installation

There are a few go-kafka clients but i prefer the client from confluent

The Go client, called confluent-kafka-go, is distributed via[ GitHub](https://github.com/confluentinc/confluent-kafka-go) and[ gopkg.in](http://labix.org/gopkg.in) to pin to specific versions. The Go client uses librdkafka, the C client, internally and exposes it as Go library using[ cgo](https://golang.org/cmd/cgo/). No separate installation of librdkafka is required for the supported platforms (Linux (glibc and musl based), and Mac OSX).

For other platforms the following instructions still apply: To install the Go client, first install[ the C client](https://docs.confluent.io/clients-librdkafka/current/index.html) including its development package as well as a C build toolchain including pkg-config.

On Debian-based distributions, install the following in addition to librdkafka:


```
sudo apt-get install build-essential pkg-config git
```


On macOS using[ Homebrew](http://brew.sh/), install the following:


```
brew install pkg-config git
```


Next, use `go get` to install the library:


```
go get github.com/confluentinc/confluent-kafka-go/kafka

```



## Hello world go kafka client

For Hello World examples of Kafka clients in Go, see[ Go-Kafka](https://docs.confluent.io/platform/current/tutorials/examples/clients/docs/go.html#client-examples-go). there you can find examples including a producer and consumer that can connect to any Kafka cluster running on-premises or in Confluent Cloud, And play with some produced messages and consuming them.


## Kafka Producer


## Idempotent Producer

Let’s first talk about the main problem of event-produce-consume systems idempotency which is not the case in Apache Kafka because they have their own system for idempotent producers or as they say “Idempotency is the second name of Kafka”. It's a very easy solution. To stop processing a message multiple times, it must be persisted to the Kafka topic only once. During initialisation, unique ID gets assigned to the producer, which is called producer ID or PID.

PID and a sequence number is bundled together with the message and sent to the broker. As sequence number starts from zero and is monotonically increasing, a Broker will only accept the message if the sequence number of the message is exactly one greater than the last committed message from that PID/TopicPartition pair. When it is not the case, the producer resends the message.


## Initialization of Kafka Producer

Initialization of producer is very easy,  you just need to call the NewProducer function.

The `NewProducer()` function expects a go map which is called `ConfigMap` where you can put your keys and values.

Since** Go**, object-oriented patterns are useful for structuring a program in a clear and understandable way, the part with mapping the Config is that I don’t prefere and I hope they will structure the Config in next versions.

Example of initialization


```Go
import (
    "github.com/confluentinc/confluent-kafka-go/kafka"
)

p, err := kafka.NewProducer(&kafka.ConfigMap{
    "bootstrap.servers": "host1:9092,host2:9092",
    "client.id": socket.gethostname(),
    "acks": "all"})

if err != nil {
    fmt.Printf("Failed to create producer: %s\n", err)
    os.Exit(1)
}
```



## Asynchronous Writes

In Go, you initiate a send by calling the `Produce()` method, passing a `Message` object and an optional `chan Event `that can be used to listen for the result and do some local calculations, writes and etc. The Message object contains an opaque `interface{}` field that can be used to pass arbitrary data with the message to the subsequent event handler.


```Go
deliveryChan := make(chan kafka.Event, 10000)
err = p.Produce(&kafka.Message{
    TopicPartition: kafka.TopicPartition{Topic: "topic", Partition: kafka.PartitionAny},
    Value: []byte(value)},
    deliveryChan,
)
```


This producer example shows how to invoke some code after the write has completed, you can use the delivery report channel passed to Produce to wait for the result of the message send:


```Go
e := <-deliveryChan
m := e.(*kafka.Message)

if m.TopicPartition.Error != nil {
    fmt.Printf("Delivery failed: %v\n", m.TopicPartition.Error)
} else {
    fmt.Printf("Delivered message to topic %s [%d] at offset %v\n",
               *m.TopicPartition.Topic, m.TopicPartition.Partition, m.TopicPartition.Offset)
}

close(deliveryChan)
```



## Synchronous writes

**Making writes synchronous is typically a bad idea since it kills throughput**, so librdkafka is async by nature. ... Because of the different hash functions, a message produced by a Go client and a message produced by a librdkafka client may be assigned to different partitions even with the same partition key. So I prefer to stick to the async writes.

But still here it is how to make synchronous writes ,receiving from the delivery channel passed to the Produce() method call:


```Go
deliveryChan := make(chan kafka.Event, 10000)
err = p.Produce(&kafka.Message{
    TopicPartition: kafka.TopicPartition{Topic: "topic", Partition: kafka.PartitionAny},
    Value: []byte(value)},
    delivery_chan
)

 e := <-deliveryChan
 m := e.(*kafka.Message)
```


Or, to wait for all messages to be acknowledged, use the `Flush() Method.`


```Go
p.Flush()
```


You need to be careful with Flush method because it will only flush messages from the producer’s `Events() `not the ones in the` deliveryChan `and if the` Flush() ` is called and no goroutine is processing the delivery channel, its buffer will fill up and cause a timeout.


## Kafka Consumer


## Initialization of Kafka Consumer

Same as producer Go client uses a ConfigMap object to pass configuration to the consumer:


```Go
import (
    "github.com/confluentinc/confluent-kafka-go/kafka"
)

consumer, err := kafka.NewConsumer(&kafka.ConfigMap{
     "bootstrap.servers":    "host1:9092,host2:9092",
     "group.id":             "foo",
     "auto.offset.reset":    "smallest"})
```



## Retrying consumer architecture in the Apache Kafka

Message processing is real problem for systems like Apache Kafka

Implementation of a consumer that processes messages immediately just after receiving them from the Kafka topic is very straightforward. Unfortunately, the reality is much more complicated and the message processing might fail because of various reasons. Some of those reasons are permanent problems, like failure on the database constraint or invalid message format. Others, like temporary unavailability of a dependent system that is involved in message handling, can be resolved in the future. In those cases retrying of the message processing might be a valid solution.

So you can implement your simple logic for retrying messages on some type of errors.

## Delivery guarantees

I mentioned above for idempotent producers also you need to be careful in consumer too.

Apache Kafka says that “you get “at least once” delivery since the commit follows the message processing. By changing the order, however, you can get “at most once” delivery, but you must be a little careful with the commit failure."

So committing on every message would produce a lot of overhead in practice. Apache Kafka suggests a better approach would be to collect a batch of messages, execute the synchronous commit, and then process the messages only if the commit succeeded.


## Synchronous commits

The Go client provides a synchronous `Commit()` method call. Other variants of commit methods also accept a list of offsets to commit or a `Message` in order to commit offsets relative to a consumed message.

The one thing that is important when using manual offset is when you initialize the consumer you need to set the key in map “enable.auto.commit” to false.

// code snippet from confluent docs how to consume synchronous commits.


```Go
msg_count := 0
for run == true {
	ev := consumer.Poll(0)
	switch e := ev.(type) {
	case *kafka.Message:
    	msg_count += 1
    	if msg_count % MIN_COMMIT_COUNT == 0 {
        	consumer.Commit()
    	}
    	fmt.Printf("%% Message on %s:\n%s\n",
        	e.TopicPartition, string(e.Value))

	case kafka.PartitionEOF:
    	fmt.Printf("%% Reached %v\n", e)
	case kafka.Error:
    	fmt.Fprintf(os.Stderr, "%% Error: %v\n", e)
    	run = false
	default:
    	fmt.Printf("Ignored %v\n", e)
	}
}
```


**MIN_COMMIT_COUNT - is a global var for when synchronous commit is triggered.**


## Asynchronous commits

For asynchronous commits you just need simply to execute the commit  in a [Goroutine](https://gobyexample.com/goroutines) function.


## Poll Loop

Once you initialize the consumer you need to register the consumer to listen for messages on some “polls”.

Simply you need to use the builtin function Consumer.SubscribeTopics() controls which topics will be fetched in the poll.

// code snippet from confluent for registering consumer to a poll and retrieving the messages from the poll


```Go
err = consumer.SubscribeTopics(topics, nil)

for run == true {
    ev := consumer.Poll(0)
    switch e := ev.(type) {
    case *kafka.Message:
        fmt.Printf("%% Message on %s:\n%s\n",
            e.TopicPartition, string(e.Value))
    case kafka.PartitionEOF:
        fmt.Printf("%% Reached %v\n", e)
    case kafka.Error:
        fmt.Fprintf(os.Stderr, "%% Error: %v\n", e)
        run = false
    default:
        fmt.Printf("Ignored %v\n", e)
    }
}

consumer.Close()
```


**Remember you always need to `Close()` after finishing with the consumer & messages.**

Feel free to contact me if you have any questions about Go or Apache Kafka.

**References:**



*   **[Idempotent Consumer](https://camel.apache.org/components/latest/eips/idempotentConsumer-eip.html)**
*   **[Apache Kafka Idempotent Producer - Avoiding message duplication](https://www.cloudkarafka.com/blog/2019-04-10-apache-kafka-idempotent-producer-avoiding-message-duplication.html)**
*   **[Apache Kafka](https://kafka.apache.org/documentation/)**