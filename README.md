redisq
======

RedisQ - Java library for Reliable Delivery Pub/Sub over Redis

What is this?
-------------

RedisQ is a Java implementation of a queue that uses Redis as a backend. It has the following features:

 - **Multiple consumers per queue**: Each queue can have multiple consumer clients consuming messages at their own
 rate.
 - **Single or multi-threaded consumers**: Each consumer can be either single threaded or multi threaded.
 - **Distributed processing**: Multiple clients/processes/nodes can consume messages from a queue in parallel.
 - **Reliable delivery**: Each consumer on a queue will receive each message.
 - **Pluggable queue/dequeue algorithms**: By default, FIFO is used, but this is pluggable (see below).
 - **Sequential delivery**: Optionally, consumers can be configured to use a locking mechanism on a queue
 to make sure each message is delivered in order.
 - **Configurable payload serialization**: Out of the box, JAXB XML, JSON and String serializers are available.
 - **Optional pluggable retry strategies**: By default, consumers do not retry consumption. A pluggable mechanism
 exists to enable retry schemes when consuming messages.
 - **High performance**: Hey, it's Redis!

High level concepts
-------------------

#### Message queue

The core concept of RedisQ is the queue itself. A queue has a name, and that's pretty much it.
Messages can be published and consumed from a MessageQueue.
The `MessageQueue` interface provides some monitoring-like operations like getting the list of registered
consumers on the queue, the number of messages in the queue for each consumer, etc.

#### Message

A `Message` is the entity that gets published and consumed on the queue. A `Message` instance provides
some meta information about your actual message, along with its 'payload'. The payload is the actual content
that you publish and consume. In Redis, each message is stored as a Hash containing all of the message attributes.
Each attribute in the hash is stored as strings, including the payload. For this reason, a (configurable)
serialization mechanism exists. More on that later.

#### Message producer

The `MessageProducer` is the side of the system that publishes messages on a queue for consumption by
consumers. Multiple producers can exist for the same queue.

#### Message consumer

A message consumer will consume messages from the queue and pass them out to your application using the `MessageListener`
that you define (more below).

#### Consumer ID

You can define an ID for each logical application consuming messages on a queue, and messages submitted to a queue
will be distributed independently to each logical consumer. This allows for per-consumer reliable delivery of messages.
In practice, a separate Redis List is created and managed for each registered consumer ID.

Multiple application instances (or processes) can be defined using the same consumer ID for distributed processing
of messages - effectively enabling reliable clustering on your application.

Using consumer IDs is optional. If not defined, a default consumer ID is used (`default`) on both the producer
and the consumer side.

The class that is used for defining a message consumer is conveniently called `MessageConsumer`.

#### Message listener

On the consumer side, the `MessageListener` interface represents the link between the queue and your application.
Your application must implement the `MessageListener` interface in order to actually consume messages. This interface
defines a single `onMessage(Message<T> message)` method that gets called when there's a message available for consumption.
This interface is genericly typed and the type you define in your implementation actually gets passed as a hint to your
configured `PayloadSerializer`.

Configuring payload serialization
---------------------------------

By default, JAXB is used to serialize message payloads that you publish through the `MessageProducer` interface
(producer-side) and that you consume through the `MessageListener` interface (consumer side).

To change this default implementation, you  need to define a bean that implements the `PayloadSerializer` interface
in your Spring context.

A few serializers are available out-of-the-box:

 - **JaxbPayloadSerializer**: Uses JAXB to serialize your payload objects. Your payload objects *must* be annotated
 using JAXB annotations (such as `@XmlRootElement` and the like). This serializer supports inheritance in your payload
 objects.
 - **GsonPayloadSerializer**: Uses Google's GSON library to serialize your payload objects. This serializer does *not*
 support inheritance in your payload objects, but provides a simple and effective way to serialize simple message objects.
 - **StringPayloadSerializer**: Expect your payload to be a String, formatted as needed.

Maven dependency
----------------

Note: This project isn't available on Maven Central (yet).

Usage with Spring
-----------------

Note: the examples below assume you're using Spring's autowiring features.

First declare the base beans for Redis connectivity:

``` xml
    <bean id="jedisConnectionFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
        <property name="hostName" value="localhost"/>
        <property name="port" value="6379"/>
    </bean>

    <bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
        <property name="connectionFactory" ref="jedisConnectionFactory"/>
        <property name="keySerializer">
            <bean class="org.springframework.data.redis.serializer.StringRedisSerializer"/>
        </property>
    </bean>

    <bean id="redisOps" class="ca.radiant3.redisq.persistence.RedisOps" />
```

Then declare each queue as a bean of type RedisMessageQueue:

``` xml
    <bean id="myQueue" class="ca.radiant3.redisq.RedisMessageQueue">
        <property name="queueName" value="my.queue"/>
    </bean>
```

Once your queue bean is created, you need to attach a Producer:

``` xml
    <bean id="messageProducer" class="ca.radiant3.redisq.producer.MessageProducerImpl">
        <property name="queue" ref="myQueue"/>
    </bean>
```

and/or a Consumer:

``` xml
    <bean id="messageListener" class="..."/>

    <bean id="messageConsumer" class="ca.radiant3.redisq.consumer.MessageConsumer">
        <property name="queue" ref="myQueue" />
        <property name="consumerId" value="someConsumerId" />
        <property name="messageListener" ref="messageListener"/>
    </bean>
```

Usually, the Producer and Consumer beans will reside in distinct application and processes, but nothing
prevents you from having both a Producer and a Consumer within the same application.

Using multi-threading on consumers
----------------------------------

By default, consumers are using a threading strategy that uses a single thread. This is easily configurable using
 Spring:

``` xml
    <bean id="messageConsumer" class="ca.radiant3.redisq.consumer.MessageConsumer">
        <property name="queue" ref="myQueue" />
        <property name="consumerId" value="someConsumerId" />
        <property name="messageListener" ref="messageListener"/>
        <property name="threadingStrategy">
            <bean class="ca.radiant3.redisq.consumer.MultiThreadingStrategy">
                <constructor-arg name="numThreads" value="4"/>
            </bean>
        </property>
    </bean>
```

Publishing messages to a queue
------------------------------

Sample code (once your Spring beans are properly setup as detailed above):

``` java
    @Autowired
    private MessageProducer queue;

    ...

    queue.create(new SomePayload("with some data")).submit();
```

Consuming messages from a queue
-------------------------------

``` java
    public class SomePayloadListener implements MessageListener<SomePayload> {

        @Override
        public void onMessage(Message<SomePayload> message) {
            SomePayload payload = message.getPayload();

            // do your stuff with the payload...
        }
    }
```

Manually starting up consumers
-------------------------------

By default, instances of `MessageConsumer` will automatically start consuming messages from their queue
when the application starts up. If you want to manually control when the consumers start, set
`autoStartConsumers` to `false`  on your consumer instances:

``` xml
    <bean id="messageConsumer" class="ca.radiant3.redisq.consumer.MessageConsumer">
        <property name="queue" ref="myQueue" />
        <property name="consumerId" value="someConsumerId" />
        <property name="messageListener" ref="messageListener"/>
        <property name="autoStartConsumers" value="false"/>
    </bean>
```

Enabling retries for failed messages
-------------------------------------

RedisQ does not retry message consumptions when an exception arises. You must configure a retry strategy
on your consumers in order to enable retries. Moreover, your code must explictly throw a `RetryableMessageException`
to tell RedisQ that a consumer error has been identified, and this error is recoverable thus can be retried.

``` xml
    <bean id="messageConsumer" class="ca.radiant3.redisq.consumer.MessageConsumer">
        <property name="queue" ref="myQueue" />
        <property name="consumerId" value="someConsumerId" />
        <property name="messageListener" ref="messageListener"/>
        <property name="retryStrategy">
            <bean class="ca.radiant3.redisq.consumer.retry.MaxRetriesStrategy">
                <constructor-arg name="maxRetries" value="2"/>
            </bean>
        </property>
    </bean>
```

Two `MessageRetryStrategy` implementation are provided out-of-the-box:

- **`NoRetryStrategy`**: (default) Does not attempt any retry of messages.
- **`MaxRetriesStrategy`**: Will retry message consumption up to a configurable maximum of times.

Changing the queue/dequeue algorithm for a queue
------------------------------------------------

By default, each queue is configured to produce and consume messages using FIFO algorithm (First In First Out), but
this mechanism can be changed using the `queueDequeueStrategy` attribute on class `RedisMessageQueue`.

``` xml
    <bean id="myQueue" class="ca.radiant3.redisq.RedisMessageQueue">
        <property name="queueName" value="my.queue"/>
        <property name="queueDequeueStrategy">
            <bean class="ca.radiant3.redisq.queuing.FIFOQueueDequeueStrategy"/>
        </property>
    </bean>
```

Implementations bundled in the library (in package `ca.radiant3.redisq.queuing`):

- **FIFOQueueDequeueStrategy**: (default) Messages are submitted to the tail of a Redis List, and are consumed from the head.
- **RandomQueueDequeueStrategy**: Messages are submitted in a Redis Set, then consumed in random order from that set.
    To prevent the need for polling, a supporting Redis List is used to notify consumers of new items in the Set.


Performance tip: Disabling "multi-consumer" (fan-out) mode
---------------------------------------------------------

By default, RedisQ producers will publish messages to all registered consumers (fan-out). If your application's design does
not require multiple consumers for a given queue, then you should switch to the "single" consumer mode, this will
improve performances.

``` xml
    <bean id="messageProducer" class="ca.radiant3.redisq.producer.MessageProducerImpl">
        <property name="queue" ref="myQueue"/>
        <property name="submissionStrategy">
            <bean class="ca.radiant3.redisq.producer.SingleConsumerSubmissionStrategy"/>
        </property>
    </bean>
```

