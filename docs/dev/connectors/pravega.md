---
title: "Pravega Connector"
nav-title: Pravega
nav-parent_id: connectors
nav-pos: 6
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

* This will be replaced by the TOC
{:toc}

## Pravega Introduction
Pravega is an open-source, high-performance, durable, elastic and strongly-consistent
stream storage for continuous and unbounded data.

It provides an abstraction -- `Stream`, which stores unbounded events aligning with the design of Flink's DataStream API.
Each `Stream` is under a namespace/tenant referred as `Scope`.
Each `Stream` is split into a set of shards or partitions generally referred as `Segments` which can be auto-scaled.
Every event written to a Pravega `Stream` has an associated `Routing Key` in an append-only way into a `Segment`.
A `Routing Key` is a string used by developers to group similar Events which is the basis for event ordering guarantee.
A `StreamCut` represents a specific position in a Pravega `Stream`, which may be obtained from various API interactions with the Pravega client.

## Pravega connector
Pravega connector can be used to build end-to-end stream processing pipelines
(see [Samples](https://github.com/pravega/pravega-samples)) that use Pravega as the stream storage and message bus,
and Apache Flink for computation over the streams.

### Features & Highlights
- Exactly-once processing guarantees for both Reader and Writer, supporting end-to-end exactly-once processing pipelines
- Seamless integration with Flink's checkpoints and savepoints.
- Parallel Readers and Writers supporting high throughput and low latency processing.
- Table API support to access Pravega Streams for both Batch and Streaming use case.

### Dependency
To use this connector, please pick the add the following dependency to your project:

{% highlight xml %}
<!-- Before Pravega 0.6 -->
<dependency>
  <groupId>io.pravega</groupId>
  <artifactId>pravega-connectors-flink_2.12</artifactId>
  <version>0.5.1</version>
</dependency>

<!-- After Pravega 0.6 -->
<dependency>
  <groupId>io.pravega</groupId>
  <artifactId>pravega-connectors-flink-{{ site.version_title }}_2.12</artifactId>
  <version>0.6.0</version>
</dependency>
{% endhighlight %}

Please carefully check the Pravega version and use the appropriate version.
{{ site.version_title }} is the Flink Major-Minor version which should be provided to find the right artifact.
`2.12` is the Scala version, now only 2.12 is supported.
`0.6.0` is the Pravega version, you can find the latest release on
the [GitHub Releases page](https://github.com/pravega/pravega/releases).

Since Pravega 0.6.0, the connector support multiple Flink versions.
Pravega Flink connector maintains the most recent *three* minor version of Flink.
See below table for the supported versions.

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left">Connector release</th>
      <th class="text-left">Latest Flink version</th>
      <th class="text-left">Supported Flink versions</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0.6.0</td>
      <td>1.9</td>
      <td>1.9, 1.8, 1.7</td>
    </tr>
  </tbody>
</table>

## Deploying Pravega
There are multiple options provided for running Pravega in different environments.
Follow the instructions from [Pravega deployment](http://pravega.io/docs/latest/deployment/deployment/) to deploy a Pravega cluster.


## Usage

### Preparation

#### Configurations
A top-level config object, `PravegaConfig`, is provided to establish a Pravega context for the Flink connector.
The config object automatically configures itself from _environment variables_, _system properties_ and _program arguments_.

`PravegaConfig` information sources is given below:

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left">Setting</th>
      <th class="text-left">Environment Variable</th>
      <th class="text-left">System Property</th>
      <th class="text-left">Program Argument</th>
      <th class="text-left">Default Value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Controller URI</td>
      <td>PRAVEGA_CONTROLLER_URI</td>
      <td>pravega.controller.uri</td>
      <td>--controller</td>
      <td>tcp://localhost:9090</td>
    </tr>
    <tr>
      <td>Default Scope</td>
      <td>PRAVEGA_SCOPE</td>
      <td>pravega.scope</td>
      <td>--scope</td>
      <td></td>
    </tr>
    <tr>
      <td>Credentials</td>
      <td></td>
      <td></td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <td>Hostname Validation</td>
      <td></td>
      <td></td>
      <td></td>
      <td>true</td>
    </tr>
  </tbody>
</table>

The recommended way to create an instance of `PravegaConfig` is to pass an instance of `ParameterTool` to `fromParams`:
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ParameterTool params = ParameterTool.fromArgs(args);
PravegaConfig config = PravegaConfig.fromParams(params);
{% endhighlight %}
</div>
</div>

#### Serialization
The Pravega connector is designed to use Flink's serialization interfaces.
For example, `SimpleStringSchema` can be used to read each stream event as a UTF-8 string.

A common scenario is using Flink to process Pravega stream data produced by a non-Flink application.
The Pravega client library used by such applications defines the
[`io.pravega.client.stream.Serializer`](http://pravega.io/docs/latest/javadoc/clients/io/pravega/client/stream/Serializer.html)
interface for working with event data.
The implementations of `Serializer` directly in a Flink program via built-in adapters can be used:
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
import io.pravega.client.stream.impl.JavaSerializer;
...
DeserializationSchema<MyEvent> adapter = new PravegaDeserializationSchema<>(
    MyEvent.class, new JavaSerializer<MyEvent>());
{% endhighlight %}
</div>
</div>

### Sources

#### Streaming API
This connector provides a `FlinkPravegaReader` class to consume events from a(or multiple) Pravega stream in real-time.
See the below for an example:
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// Enable checkpoint to make state fault tolerant
env.enableCheckpointing(...);

// Define the Pravega configuration
PravegaConfig config = PravegaConfig.fromParams(params);

// Define the event deserializer
DeserializationSchema<MyClass> deserializer = ...

// Define the data stream
FlinkPravegaReader<MyClass> pravegaSource = FlinkPravegaReader.<MyClass>builder()
    .forStream(...)
    .withPravegaConfig(config)
    .withDeserializationSchema(deserializer)
    .build();
DataStream<MyClass> stream = env.addSource(pravegaSource);
{% endhighlight %}
</div>
</div>

#### Batch API
Similarly, a `FlinkPravegaInputFormat` class is offered for batch processing. It reads events of a stream as a `DataSet`.
See the below for an example:
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

// Define the input format based on a Pravega stream
FlinkPravegaInputFormat<EventType> inputFormat = FlinkPravegaInputFormat.<EventType>builder()
    .forStream(...)
    .withPravegaConfig(config)
    .withDeserializationSchema(deserializer)
    .build();

DataSource<EventType> dataSet = env.createInput(inputFormat, TypeInformation.of(EventType.class)
                                   .setParallelism(2)
{% endhighlight %}
</div>
</div>

Details on more functionality such as checkpointing, optional start, metrics and
watermark assigning can be found in the [Document](http://pravega.io/docs/latest/connectors/connectors/index.html)

### Sinks

#### Streaming API
This connector provides a `FlinkPravegaWriter` class to write events into a Pravega stream in real-time.
See the below for an example:
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// Define the Pravega configuration
PravegaConfig config = PravegaConfig.fromParams(params);

// Define the event serializer
SerializationSchema<MyClass> serializer = ...

// Define the event router for selecting the Routing Key (lambda expression can be used for simplification)
PravegaEventRouter<MyClass> router = ...

// Define the sink function
FlinkPravegaWriter<MyClass> pravegaSink = FlinkPravegaWriter.<MyClass>builder()
   .forStream(...)
   .withPravegaConfig(config)
   .withSerializationSchema(serializer)
   .withEventRouter(router)
   .withWriterMode(EXACTLY_ONCE)
   .build();

DataStream<MyClass> stream = ...
stream.addSink(pravegaSink);
{% endhighlight %}
</div>
</div>

##### Writer Modes
Writer modes relate to guarantees about the persistence of events emitted by the sink to a Pravega Stream.
The writer supports three writer modes:

1. **Best-effort** - Any write failures will be ignored hence there could be data loss.
2. **At-least-once** - All events are persisted in Pravega. Duplicate events
are possible, due to retries or in case of failure and subsequent recovery.
3. **Exactly-once** - All events are persisted in Pravega using a transactional approach integrated with the Flink checkpointing feature.
For more details, please refer to [end-to-end exactly once blog](https://flink.apache.org/features/2018/03/01/end-to-end-exactly-once-apache-flink.html)

By default, the _At-least-once_ option is enabled and use `.withWriterMode(...)` option to override the value.

#### Batch API
Similarly, a `FlinkPravegaOutputFormat` class is offered for batch processing.
It can be supplied to make a Pravega Stream as a sink of a `DataSet`.
See the below for an example:
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// Define the output format based on a Pravega stream
FlinkPravegaOutputFormat<EventType> outputFormat = FlinkPravegaOutputFormat.<EventType>builder()
    .forStream(...)
    .withPravegaConfig(config)
    .withSerializationSchema(serializer)
    .withEventRouter(router)
    .build();

DataSet<EventType> dataSet = ...
dataSet.output(outputFormat);
{% endhighlight %}
</div>
</div>

Details on more functionality such as event time ordering, watermark propagation and metrics can be found in the
[Document](http://pravega.io/docs/latest/connectors/connectors/index.html)
