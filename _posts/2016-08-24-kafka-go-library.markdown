---
layout: post
title:  "First Thoughts on the Confluent Golang Kafka Library"
date:   2016-08-24 00:00:00
categories: golang kafka library sarama confluent
---

I recently took a look at [Confluent's Golang Kafka Library](https://github.com/confluentinc/confluent-kafka-go). At my day job we use Shopify's excellent Sarama library pretty extensively to both produce and consume messages from Kafka, so I thought I would compare them.

Right now `confluent-kafka-go` offers both channel and polling-based API calls for consumers and producers. This is a temporary condition while they figure out which ones to support long term. I'm happy to give my two cents on the issue. 

Building
===

Right off the bat, `confluent-kafka-go` depends on `librdkafka`, a C client library for Kafka. Cgo is a great way to reuse an existing, stable codebase, but it's also a big pain - especially for multi-platform builds. Having a non-Go dependency makes setting up developer and build machines that much more complicated. On Windows [we use mingw](http://www.agardner.me/golang/windows/cgo/2015/09/07/go-windows-linking.html) to build cgo code, but `librdkafka` only seems to include build scripts for Visual Studio. I'm sure we could make it work, but with Sarama's pure-Go implementation this isn't an issue.

Consumer API
===

In my opinion using channels for the consumer interface makes perfect sense:

- Using `select` statements for this kind of input is natural.
- The `Events` channel could be private, with an accessor method that gets a read-only channel. This would make the API clearer: 
        func (p *Producer) Event() <-chan Event
- Putting all the different event types - errors, rebalances, and incoming Kafka messages - makes the ordering clearer. Sarama splits errors and incoming messages across multiple channels, but I like this better.
- I wish there were a low-level consumer API as well. We do our own offset management, which definitely puts us in the minority of Kafka users.

Producer API
===

For the producer interface, channels seem clumsy - I prefer calling `Produce` and then waiting for the result on a callback channel:

- Just like the consumer, it would be clearer to have getters for the `Event` and `ProduceChannel` channels. 
- Making the user build the `Message` struct is not great. As the struct definition changes with time this could cause unexpected behaviour due to zero values. I'd prefer clearer `Produce` methods, like:

        func (p *Producer) Produce(topic string, key []byte, value []byte, opaque interface{}) 

- Following up on the last point, it would be great if _some_ of the `Produce` methods handled partition assignment based on a configurable rule - consistent hashing, round-robin, etc. 
- I'm really glad there's an explicit `Flush()` method. 
- It might be possible to improve performance by exposing a `ProduceBatch` method which accepts a list of `Message`s which all share the same `delivery_chan` and `opaque` to reduce the number of cgo round trips and possible contention for the `cgo_lock`.

Closing Thoughts
===

`confluent-kafka-go` is an interesting project: using `librdkafka` means new Kafka features should be available sooner. There may also be perfomance benefits - I plan to do some benchmarking and report back. Conversely, the use of cgo makes it more complicated to build and might rule it out for some users. 
