---
layout: post
title: Building a centralized logging system. Choosing the architecture.
tags: [logging, realtime, bigd]
---

It's hard to imagine any decent IT project without a logging system. I tend to look into logs even on my own laptop to find **why** something is not working. Let alone, if you have a _service_ which generates millions of dollars per day. If it's down, you're losing money. So you need to make sure you're able to find the problem as quickly as possible if something goes wrong.

Your infrastructure could have tens, hundreds or even thousands of hosts. Imagine the time and effort it would take to log into each of your hosts and check the logs in hope of finding where the problem occurred. And while our service is down, our lost time converts to lost money. What you need in this case is a **centralized logging system** that would store all the logs in one place, make it easy to search through them and analyze.


 _Remember logs are your go-to buddy when a service goes down and you want to know what happened and why_.<!--break-->

Logging is closely related to the monitoring. It gives you the insight on what is going on inside your infrastructure and can even send you alerts when something bad happens.

This is the **first** post of a new series I want to dedicate to _exploring the options of building a centralized logging system with existing opensource tools_. Specifically, for the systems with *a high flow rate of log messages*.

There are of course many _commercial solutions_ like Splunk, Sumo Logic. But these could be very expensive even for such big companies like LinkedIn where they use ELK stack. And generally if you read through different blogs and forums, you'll see how many people complain that their commercial logging system costs them a fortune. So we'll focus on opensource tools which let you build (with a little bit of effort) your own centralized logging system.  

Before we start, I just want to have a quick look at the criterias for a logging system. These are the criterias that names Tin Lee (Senior SRE at LinkedIn) in his [presentation](https://www.youtube.com/watch?v=51wKfT4t7Jg) at ELK Meetup:

* ```Scalable``` - horizontally, because the volume of logs will grow as grow your infrastructure.
* ```Close to real-time``` - if it takes hours to get the result you want, it becomes useless.
* ```Inexpensive``` - no company wants to spend too much money on logging.
* ```Flexible```  - once you install a solution you can find many other ways of using it that you didn't think of in the     beginning, so it has to be adaptable.
* ```Large user community``` - which means support.
* ```Opensource``` - have control over the code.

These criterias have been chosen at LinkedIn where they collect tremendous amounts of logs each day. They seem sound to me, and I generally tend to keep these in mind when working on a logging system with *a high flow rate of log messages*.

If you're already familiar with logging systems, you know that there is a great variety of tools out there that you can use for logging. Yet, you'll often find people use their own self-written tools for managing logs. And it doesn't seem so surprising, if you understand that every infrastructure you are going to set up a logging system for is _unique_. So there is no specific pattern or a set of tools that you can use for just any infrastructure. Instead, there are plenty of those patterns and tools and choosing the right one depends on your needs and type of infrastructure. And still, until you try a specific tool or pattern, you'll never know if it suites you or not. This is why I call it here _"building"_ a logging system as a _continuous process_ of creating a centralized logging system that will be effective enough for your particular case.

Here, in the first part of this blog series, we start with choosing an architecture of a logging system. In the following posts, we'll take a closer look at some of the opensource tools we can use at each level of the architecture we've chosen.

Get yourself familiar with the terms we will use:
* ```Log Shipper (or Log Forwarder)``` - collects log data and sends it to the ```Log Indexer``` for processing
* ```Log Indexer (or Log Collector/Aggregator)``` - _aggregates_  logs from different sources while performing transformations like Grok and indexing into storage. Ideally, they should provide a [Unified Logging Layer](http://www.fluentd.org/blog/unified-logging-layer) for you logging system.

Let's start with a simple case, when you have a relatively small system  which doesn't generate that much logs, say 300-500 msg/sec, including application and system logs. Then architecture of your logging system would look more or less like this.
![800x400](/public/img/logging/logging-architecture0.jpg)
Architecture like this is quite simple and used extensively for systems with relatively low log rate. As you can see on the picture, you would usually use shippers (Rsyslog, Filebeat, etc.) which simply read local log files for the most cases and send log data to a log indexer which in turn would process these logs and persist them to storage. Besides, some services might send logs directly to the indexer. For instance, your application could be writing logs directly to the Logstash or Graylog which are some of the commonly used indexers.

_Log shippers_ get installed on the host where logs are generated. They are lightweight agents whose main function is to transport logs from hosts to _log indexer_ which does all the log processing and transformation. Obviously, depending on how complex the processing is, it can consume a lot of memory. So you usually want to install log indexer on a separate host.

But what if the log flow rate is 15k msg/sec or even bigger? Will our single indexer be able to handle such a load without crashing, loosing messages, and still provide us a real time logging system we want? In this case, we would normally add a few more indexer instances to handle the log traffic.

When we do this, we start thinking about load balancing between log indexers. Some shippers (like [Filebeat](https://www.elastic.co/guide/en/beats/filebeat/current/load-balancing.html)) can be configured to balance the load between indexers. A [load balancer](http://docs.graylog.org/en/latest/pages/architecture.html#bigger-production-setup) like HAProxy could also be used.

Another option would be to use a **message broker**, which can provide us with the load balancing we're seeking as well as as to give us a **buffer** for logs. A message broker simply queues log messages from shippers and makes them available for log indexers.

Why do we need this buffer? Here's a just a few reasons why.
1. **Event spikes**

    Log flow rates are very hard to predict. When we say we have 15k msg/sec, this doesn't mean that we actually have 15k msg/sec all the time, it's an average number. Instead, what happens is that sometimes log flow rate suddenly increases. For example, when you deploy your application with a bug. This sudden and excessive logging is known as a _**spike**_.

   Having a message broker as a buffer, we can protect log indexers and storage from this burst of data. Indexers like Logstash and Graylog allow throttling of the input when used with a message broker, so we can make sure they handle the log traffic at their normal pace without failure.
2. **Indexers or Storage is not reachable**

    Imagine a situation when our indexers or storage suddenly go down. We cannot afford to stop our application, because this means we start loosing money, but neither do we want to start loosing logs. A message broker comes to the rescue. When a situation like this happens, we continue to stream log data into a message broker and hold them there until our indexer or storage comes back up.

    A few examples when it might happen. If you use Elasticsearch as a storage, upgrading it requires a full cluster restart. And simply, your network could be down.  

Some of the log shippers allow local buffering, which means when your application is generating more logs that your indexers can ingest, a shipper slows the traffic down and starts to buffer log messages either in memory or to a file. Although, you have to consider that in this case your local filesystem will become this buffer and you will have to make sure enough disk space or memory exists to house all these logs.

Thus, we are coming to this architecture
![800x400](/public/img/logging/logging-architecture1.jpg)
This is the architecture we’re going to refer to in the future posts when we explore the opensource options at each level.
I want to note that in this schema _Broker_ is not necessarily a separate host or a service like Kafka or Redis, but it can also be any mechanism that a shipper could use to avoid overloading a _log indexer_. So it could be local buffering we talked about earlier, although in systems with a _very high flow log rate_ it is usually separate hosts with message queue servers.

I also would like to note that the problem we are facing by adding a new component is its integration with the rest of the components of our architecture. Not everything can write logs to a message broker we choose and we have to be aware of that.

To mitigate the problem with compatibility of different message brokers and shippers, I also like the idea of adding a central _log shipper_ like Logstash. One of the reasons why the Logstash is so awesome and popular is that it supports almost every input you can think of and has almost every output, so it can fit well as central log shipper. Then, for example, you will be able to collect system logs from your network devices (like [Cisco ASA](https://jackhanington.com/blog/2015/06/16/send-cisco-asa-syslogs-to-elasticsearch-using-logstash/)) because Logstash has input plugin for syslog.
![800x400](/public/img/logging/logging-architecture2.jpg)

**Conclusion:** in this post we talked about the importance of having a centralized logging system, got ourselves familiar with the logging terms and chose an architecture of a logging system that we're going to use to explore our options at each level.
