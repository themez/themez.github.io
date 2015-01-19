---
layout: post
title: RabbitMQ In Practice
category: tech
tags: [RabbitMQ]
---


We've bean using RabbitMQ as our message exchange in payment module for more than one year from January last year. It's famous on its good performance and stability as an message broker, today I'm not gonna give yet another prove to its performance. I want to talk something on engineering practice, how to ensure not a single messages not lost, how to make use of it's core features, and also build missing features or facilities which are not shipped within the package. Give a conclusion from the practice in the last year.

----------

Basic Exchanges
---
Actually there are 4 kinds of exchanges defined in amqp protocal.

 - Direct exchange: routing messages to some queue has the same name as the routing key.
 - Fanout exchange: routing messages to all queues bound to it and ignore the routing key.
 - Topic exchange: routing messages to all queues bound to it with matched routing key pattern.
 - Header exchange: routing messages base upon their header values.
 
The topic exchange has broader set of use cases, cause the binding routing key can be a specific string or pattern, and can routed to one or many queues, listeners can get messages which they need and drop those they don't care. And we can make a tracing listener base on topic exchange, for example, if we want to trace all order processing messages, which has routing keys start with "order", then the tracing queue can bound to the topic exchange with routing key `order.#`(* match one word and # match any number words).


Dead Letter Exchange
---
In a payment system, every business request should be take care appropriately, if a request cannot be processed correctly at a time, there should be retry strategy, if it is a DB connection timeout or something runtime failure, it might be processed correctly after one or two times' retry, [spring retry](https://github.com/spring-projects/spring-retry)  would do the work. But what if there's a logical bug that make the order in a state cannot be handled? The same request would fail and fail again, it's not possible to make a retry strategy to handle all kind of error. Some times a human has to be involved, a good practice is to utilize the dead letter feature of RabbitMQ. 

A dead letter exchange is no different to a normal exchange, any exchange can be set as dead letter exchange to a queue, after that, any message rejected from the queue would go to the dead letter exchange, then routed to corresponding dead letter queue, you can make your error handler listening on the dead letter queue, do auto retry according to a retry strategy, or notify a human to involve after fall back. All these processes can be easily decoupled from main service.

Confirms and Returns
---
Normally a publisher publish a message with a routing key to some exchange, then the Rabbit broker would do it's part which publisher wouldn't care any more. But in some case the publisher want to make sure the message has been correctly routed to some queue, then we need publisher confirms. When the message been routed to a queue(for non durable messages), been consumed or been persisted(for durable messages), at the first time, the publisher got a confirm, if the message routed to nowhere, the publisher got a return. Here's more detail: [http://www.rabbitmq.com/blog/2011/02/10/introducing-publisher-confirms/](http://www.rabbitmq.com/blog/2011/02/10/introducing-publisher-confirms/). Normally we don't need this feature if we only use durable message and all queues and exchanges is correctly declared.

Selective Consuming
---
RabbitMQ doesn't provide selective receiving message feature directly, a consumer must fetch all messages and drop those it doesn't want to receive.
Sometimes a developer want to debug a message handling process in test environment, he can start the application and listen on the test queue, if do so, it cannot be sure which application would receive the message. 
Or one can declare it's own queue and bind to the same exchange, he's own and the test environment applications all got the same message, but here comes another problem, what if the message handling process cause an state change and cannot be done twice, so the test env application must drop the message to let the dev application handle it.
In our case, we want to achieve that if the payment request come from a dev machine, then the async response should go to the same machine. It's not hard if utilize zookeeper: when the dev application make a payment request, it register the order with it's unique identity(for example, IP) on zookeeper as a ephemeral node, what the test machine received the response, it checks on zookeeper if the order been preserved, if yes, it just drop the messsage to let a dev application process it.

Management Plugin
---
TBD

Spring Rabbit framework
---
TBD
