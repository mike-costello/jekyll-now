---
layout: post
title: Use CDC to create AMQP Based Events with Apache Camel and Debezium
tags: CDC Debezium Integration CloudNative Knative Interconnect Camel QDR AMQP QpidDispatchRouter Kubernetes Openshift 
eye_catch: /assets/img/debezium.png
---

# Using Debezium To Create *AMQP* Based Events 

***Change Data Capture*** is an important architectural technique as we seek to describe OLTP events in our event driven architectures. *Change Data Capture* seeks to describe data written to traditional relational databases as consumable events for an event streaming platform.

**Debezium** is a way to *tail database commit logs* and create **events** out of these database writes. We use *Debezium* to ensure that as our surrounding business processes' persist transactions to OLTP Stores *(aka Databases)* that we stream these events to an event bus, where stream processors can provide further integration with this data. 

Often this event bus is ***Apache Kafka***, as we use *Debezium* with *Kafka Connect*; however, Kafka Connect has limited message transformation capabilities, and only one sink/target Kafka. 

Using **Apache Camel**, we can use *Debezium* to persist events to any number of targets, adapters, or services. Given *Apache Camel's* maturity as an enterprise integration appliance, it provides a much more mature, and feature rich environment through which to use *Debezium* for *Change Data Capture* purposes.  

In this article, we'll discuss how to create meaningful AMQP events from change data capture events, and persist them along an event bus/mesh. 

## Using Camel with embedded *Debezium*
Without getting too far into what *Apache Camel* is, it is worth pointing out that *Apache Camel* provides an integration framework with hundreds of adapters. *Apache Camel* may also be run as a microservice self-managed, via other runtimes such as Springboot, Vert.x, or OSGI, and fits well as a runtime for *Debezium* providing out of the box support for many *Debezium* sources. 

In this article, we will leverage the work done in a great article [Integration Scenarious With Debezium and Camel](https://debezium.io/blog/2020/02/19/debezium-camel-integration/). 

We'll extend the work done in this article to do a few more/different things: 
* Transform our payload to meet the OpenAPI specification 
* Prepare our payload for use as an AMQP based event 
* Prepare an event mesh with *Apache Qpid Dispatch Router* 
* Consume this message as an AMQP peer to our *Debezium* runtime 

To illustrate this examples we'll be using the source code found here: [Camel Debezium Examples](https://github.com/mike-costello/camel-debezium-examples)

### Wiring up Camel for *Debezium* Usage

Initally, there are a few things we'll need to do: 
* We'll need to ensure we're using Apache Camel 3.x
* We need to wire up our camel route to use the Camel Posgres Debezium component [Camel Postgres Debezium Component] (https://camel.apache.org/components/latest/debezium-postgres-component.html)

```
<dependency>
    <groupId>org.apache.camel</groupId>
    <artifactId>camel-debezium-postgres</artifactId>
    <version>x.x.x</version>
    <!-- use the same version as your Camel core version -->
</dependency>
```

Using the Debezium Camel component for Postgres, we can wire up a camel route like: 

```

@Override
	public void configure() throws Exception {
		
		from("debezium-postgres:localhost?"
                + "databaseHostname={{database.hostname}}"
                + "&databasePort={{database.port}}"
                + "&databaseUser={{database.user}}"
                + "&databasePassword={{database.password}}"
                + "&databaseDbname=postgres"
                + "&databaseServerName=localhost"
                + "&pluginName=pgoutput"
                + "&schemaWhitelist={{database.schema}}"
                + "&tableWhitelist={{database.schema}}.making_tests"
                + "&offsetStorage=org.apache.kafka.connect.storage.MemoryOffsetBackingStore")
                .routeId("debeziumPGRoute")
                .routeDescription("This route  consumes from a PostGres DB and persists an AMQP Event")

```

#### Debezium Configuration 

Its worth noting a few things about our *Debezium* usage. We wire up for our database with the usual suspects hostname, port, etc., and we wire up *Debezium* to whitelist a schema and table, which Debezium will use the underlying Postgres logical decoding of *pgoutput*. *Debezium* will create a struct out of this of the form: 

```
org.apache.kafka.connect.data.Struct
```

which is a Kafka Connect data structure that we pull in with the Camel Debezium component. It is worth noting the use of the *in memory backing store* for *Debezium*. *Debezium* offers a [FileOffsetBackingStore](https://github.com/a0x8o/kafka/blob/master/connect/runtime/src/main/java/org/apache/kafka/connect/storage/FileOffsetBackingStore.java) as well which may more sense for persistent needs and a closer to production ready deployment. 

#### The rest of the Camel route

````
                .log(LoggingLevel.INFO, "Incoming message ${body} with headers ${headers}")
                .convertBodyTo(TestEvent.class)
                .setHeader("CamelInfinispanKey", simple("${body.id}"))
                .setHeader("CamelInfinispanValue", simple("${body}"))
                .setHeader("CamelInfinispanOperation", simple(InfinispanOperation.PUTIFABSENT.toString()))
                .setHeader("eventPayload", simple("${body}"))
                .to("infinispan:pg.event")
                .setBody(simple("${header.eventPayload}"))
                .doTry()
                	.to("amqp:{{database.schema}}.dbevents?connectionFactory=connFact")
                .doCatch(IOException.class)
                	.log("** Exception thrown: ${exception.message} **")
                	.log("Body of the message: ${body}")
                	.throwException(new Exception("${exception.message}"))
                .endDoTry();
		
````
The rest of this route simply leverages our Camel stream handling capabilities by using a Camel Converter, storing the message to Infinispan, and inevitably emitting the message via an AMQP message sender all wired up by Camel. As Camel is an incredibly mature integration framework, nearly any number of transformation, service mediation, or other enteprise integration patterns may be leveraged at this point with literally hundreds of adapters. 

While this code snippet would need a good deal of elaboration to be ready for a real CDC use case, it displays that creation of events (even those data ingress events that might originate from more traditional legacy OLTP approaches) need not be coupled to Kafka as a store.

## AMQP 1.0 as the Event Mesh Transport  

By maintaining Kafka as a central bus for our event mesh, we suffer from lack of the following capabilities: 
### * *Event routing* 
While there are certainly a number of ways to route Kafka producers/consumers using proxies, load balancers, LTM/GTM approaches, etc. and other machinations, event producers and consumers are coupled to the underlying Kafka wire protocol and inevitably Kafka broker. This coupling implies a request/response contract with the underlying broker, and bakes in Kafka specific needs to event producers. 

### * *Built-In Settlement* 
While Kafka clients can certainly be written to avail themselves of various QoS by employing various techniques, emitting events to Kafka as a central event bus implies that a client is bound to Kafka's In Sync Replica conditions. While this might work well in a QoS 0 type of situation where a client does not block and continues processing while spinning out a thread to handle the interaction with Kafka, disposition of the event by the event receiver is still bound by Kafka's broker level ISR configuration. In the case of many event types, this monolithic behaviour to determine disposition of the event emitted, is inappropriate for the event producers use case.

### * *Multi-Tenancy*  
Additionally, Kafka has no real notion of multi-tenancy making it imperfect for use as an event bus. While we can certainly use keys for our message payloads that imply multi-tenancy in our broker, by leveraging an event mesh such as ***Qpid Dispatch Router*** [QPID Dispatch Router](https://qpid.apache.org/components/dispatch-router/index.html), we can create a truly multi-tenany event mesh. For more information on how to leverage a vhost so that policy may be applied in a multi-tenant fashion to our event mesh, please check out: [Configuring Authorization and Vhosts in QDR](https://qpid.apache.org/releases/qpid-dispatch-1.11.0/user-guide/index.html#configuring-authorization-qdr)

### * *Producer Flow Control* 
The greatest advantage of decoupling Kafka as a store from our event mesh, is that the Kafka wire protocol is immature in its capabilities. While we do have flow control provided us to by our Kafka brokers, this approach is ad hoc and happens after the fact. It is not until we have exhausted our in memory backing queue in the broker that we will have tripped thresholds for flow controlling producers. 

As a result, while emmitting our events directly to Kafka neatly couples our events to our store, and we leverage Kafka Connect to tail the head of commit logs in a DB that allows us to create a single topic producer, we have no way of decoupling a particular message producer from all message producers in the way that they are being flow controlled. As our number of producers grow, Kafka presents us little capability to dissagregate these stream producers from the overall broker capacity. 
 
AMQP 1.0 makes this capability a Layer 7 wire protocol capability between peers. For instance, if the AMQP event receiver that receives our AMQP event produced from this code sample is not capable of taking in messages, it will offer us 0 credit (AMQP 1.0 employs a credit cased flow control mechanism) and block our producers. This implies that an individual CDC emitter may be flow controlled by our AMQP event sink. 

### * *Event Routing* 
While there are certainly a number of ways to route Kafka producers/consumers using proxies, load balancers, LTM/GTM approaches, etc. and other machinations, event producers and consumers are coupled to the underlying Kafka wire protocol and inevitably Kafka broker. This coupling implies a request/response contract with the underlying broker, and bakes in Kafka specific needs to event producers. 
