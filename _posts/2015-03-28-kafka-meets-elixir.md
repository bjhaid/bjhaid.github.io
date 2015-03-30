---
layout: post
title: Kafka meets Elixir
excerpt: Exploring how to integrate with Kafka from Elixir.
modified: 2015-03-28
tags: [elixir, kafka]
comments: false
---

[Apache Kafka](http://kafka.apache.org/) is a high-throughput distributed messaging system.  

&nbsp;  
It is:  

- Fast (A single Kafka broker can handle hundreds of megabytes of reads and writes per second from thousands of clients);
- Scalable (Kafka is designed to allow a single cluster to serve as the central data backbone for a large organization. It can be elastically and transparently expanded without downtime. Data streams are partitioned and spread over a cluster of machines to allow data streams larger than the capability of any single machine and to allow clusters of co-ordinated consumers);
- Durable (Messages are persisted on disk and replicated within the cluster to prevent data loss. Each broker can handle terabytes of messages without performance impact);
- Distributed by Design (Kafka has a modern cluster-centric design that offers strong durability and fault-tolerance guarantees).[1]

This post intends to explore how to integrate with Kafka from Elixir, the post assumes you have some knowledge of Elixir if not checkout the [Elixir getting started](http://elixir-lang.org/getting-started/introduction.html).

### Setting up Kafka
We would set up Kafka from docker images (for production use consider [http://kafka.apache.org/documentation.html#quickstart](http://kafka.apache.org/documentation.html#quickstart)) using [fig](http://www.fig.sh/install.html)

{% highlight bash %}
$ git clone git@github.com:bjhaid/kafka_docker.git
$ cd kafka-docker/
$ fig up
{% endhighlight %}

We should have our kafka/zookeeper nodes up and running.

### Create Kafka project

{% highlight bash %}
$ mix new kafka_ex_demo
$ cd kafka_ex_demo/
{% endhighlight %}

Edit the mix.exs as below:

{% highlight elixir %}
...

 def application do
   [applications: [:logger, :kafka_ex]]
 end

 defp deps do
   [
     {:kafka_ex, "~> 0.0.2"},
   ]
 end

...
{% endhighlight %}

Get `kafka_ex`

{% highlight bash %}
$ mix deps.get
{% endhighlight %}

Update the application config (config/config.exs) to include the kafka node:

{% highlight elixir %}
use Mix.Config

config KafkaEx,
  brokers: [{"192.168.59.103", 49154}]
{% endhighlight %}

Make sure the above matches what you have in your kafka logs.

#### Producer

We would create a producer that runs in a infinite loop sleeping every 500ms and producing current time, create lib/producer.exs and add the below:

{% highlight elixir %}
producer_fn = fn ->
  helper_fun = fn(fun) ->
    KafkaEx.produce("kafka", 0, (inspect :os.timestamp))
    :timer.sleep(500)
    fun.(fun)
  end

  helper_fun.(helper_fun)
end

producer_fn.()
{% endhighlight %}

Open a new shell and run:

{% highlight bash %}
$ mix run lib/producer.exs
{% endhighlight %}

#### Metadata

To grab metadata for the topic we produced above, create lib/metadata.exs and add:
{% highlight elixir %}
IO.inspect KafkaEx.metadata(topic: "kafka")
{% endhighlight %}

Open a new shell and run:

{% highlight bash %}
$ mix run lib/metadata.exs

%{brokers: %{49154 => {"192.168.59.103", 49154}},
  topics: %{"kafka" => %{error_code: 0,
      partitions: %{0 => %{error_code: 0, isrs: [49154], leader: 49154,
          replicas: [49154]}}}}}
{% endhighlight %}

#### Consumer

To consume messages and print to console the messages published to the `kafka` topic, create lib/consumer.exs and add:

{% highlight elixir %}
IO.inspect KafkaEx.fetch("kafka", 0, 0)
{% endhighlight %}

Open a shell and run:

{% highlight bash %}
$ mix run lib/consumer.exs
{% endhighlight %}

You would get an output similar to:

{% highlight elixir %}
{:ok,
 %{"kafka" => %{0 => %{error_code: 0, hw_mark_offset: 654,
       message_set: [%{attributes: 0, crc: 2792772004, key: nil, offset: 0,
          value: "{1427, 640108, 212625}"},
        %{attributes: 0, crc: 4244189747, key: nil, offset: 1,
          value: "{1427, 640109, 250613}"},
...
{% endhighlight %}

Note `value: "{1427, 640108, 212625}",` and `value: "{1427, 640109, 250613}"` is from `inspect :os.timestamp` from the producer

#### Streaming

To stream messages from the `kafka` topic and print the message to console, create lib/stream.exs and add:

{% highlight elixir %}
KafkaEx.create_worker(:streaming_worker)
for message <- KafkaEx.stream("kafka", 0, worker_name: :streaming_worker) do
  IO.puts "Got #{inspect message}"
end
{% endhighlight %}

Open a shell and run:

{% highlight bash %}
$ mix run lib/stream.exs
{% endhighlight %}

You would get an output similar to:

{% highlight bash %}
Got %{attributes: 0, crc: 2792772004, key: nil, offset: 0, value: "{1427, 640108, 212625}"}
Got %{attributes: 0, crc: 4244189747, key: nil, offset: 1, value: "{1427, 640109, 250613}"}
Got %{attributes: 0, crc: 2678133112, key: nil, offset: 2, value: "{1427, 640109, 759012}"}
Got %{attributes: 0, crc: 1683310624, key: nil, offset: 3, value: "{1427, 640110, 271154}"}
Got %{attributes: 0, crc: 2197677395, key: nil, offset: 4, value: "{1427, 640110, 783484}"}
{% endhighlight %}

`KafkaEx.stream` implements the Enumerable protocol, so you can use it with functions from the Enum and Stream modules, this allows us to do very fancy MapReduce operations on the messages as they arrive.

#### Offsets

To fetch offsets for the `kafka` topic, create lib/offset.exs and add:
{% highlight elixir %}
IO.puts "Earliest offset is: #{inspect KafkaEx.earliest_offset("kafka", 0)}"
IO.puts "Latest offset is: #{inspect KafkaEx.latest_offset("kafka", 0)}"
{% endhighlight %}

Open a shell and run:

{% highlight bash %}
$ mix run lib/offset.exs
{% endhighlight %}

You would get an output similar to:

{% highlight bash %}
Earliest offset is: {:ok, %{"kafka" => %{0 => %{error_code: 0, offsets: [0]}}}}
Latest offset is: {:ok, %{"kafka" => %{0 => %{error_code: 0, offsets: [654]}}}}
{% endhighlight %}

The examples shown can be found [here](https://github.com/bjhaid/kafka_ex_demo).

1. [http://kafka.apache.org/](http://kafka.apache.org/)
