---
layout: post
title: >
  Building a centralized logging system. <br /> Log shippers (part 1).
tags: [logging, logship, logcol]
---
_In this part of our blog series we're going to look at the most popular opensource tools used for log shipping._

I love reading [Sematext blog](https://sematext.com/blog/). These guys are one of the few who write regularly about different logging tools and approaches to setting up a logging system. They have a post about [log shipper popularity](https://sematext.com/blog/2014/10/06/top-5-most-popular-log-shippers/) based on the poll results. Although,  it dates back to 2014, you may notice that not much changed in the world of log shippers since then and those whose names are on the picture of the poll results haven't lost their popularity to this day.

![800x400](/public/img/logging/shippers-popularity.png)  
<!--break-->
Considering the popularity of a fairly new log shipper [Filebeat](https://www.elastic.co/products/beats/filebeat) nowadays, I changed this picture a little. The Filebeat is a product of the same company that develops Logstash, so I added it next to Logstash.

We are going to take a look at each one of these _popular shippers_. I worked with some of them, others will be more or less new for me. _My goal is to learn more about these tools and summarize my knowledge about each of them._

There is [a great post](https://sematext.com/blog/2016/09/13/logstash-alternatives/) (again on Sematext) about popular shippers. I definitely recommend reading it. This is one of the sources we're going to refer to in this series of posts.  

Shippers we're going to look at:

* Logstash and Filebeat
* Fluentd
* Rsyslog
* Syslog-ng

You may wonder why _Apache Flume_ is not on the list, because it's clearly on the picture. The reason why I decided not to include Apache Flume in this list is because this shipper is built around _Hadoop ecosystem_. I'm going to write a separate post on _Apache Flume_ and _Hadoop_ later, but for now we will focus on the architecture when _Elasticsearch_ is used as a storage.

### Logstash and Filebeat

[Logstash](https://www.elastic.co/products/logstash) is a very popular and flexible tool. Its main strength is the abundance of different plugins which allow you to receive log data from almost any source with [inputs](https://www.elastic.co/guide/en/logstash/current/input-plugins.html), change the data representation of an event with [codecs](https://www.elastic.co/guide/en/logstash/current/output-plugins.html), apply various [filters](https://www.elastic.co/guide/en/logstash/master/filter-plugins.html) to your log traffic and send the log data further to almost every possible destination with [outputs](https://www.elastic.co/guide/en/logstash/current/output-plugins.html).

As you can see, Logstash is very powerful and would fit great as a _log indexer_ which takes the log data from different sources, does the log processing we need, and stores it to any location we want.

The Logstash's weakness has always been performance and resource consumption with the default heap size of 1GB. We certainly wouldn't want to install it on our small instances. In [this](https://sematext.com/blog/2016/09/13/logstash-alternatives/) post on Sematext, it says that based on benchmarks they did Logstash also turned out to be a lot slower than its alternatives like Rsyslog and Filebeat.

The [Logstash Forwarder](https://github.com/elastic/logstash-forwarder) was introduced to address the problems of the Logstash as a shipper. The idea was to split the log shipment and heavy log processing in two separate stages. A log shipper would be a lightweight agent, so we could install it on any host. All it would have to do is to take the log data and send it to the Logstash. It still could do some processing and data formatting _to make the processing easier for log indexer_. A Logstash in turn would be installed on a standalone host with enough resources to perform the log processing and transformation we need and persist logs into storage.

[Filebeat](https://www.elastic.co/products/beats/filebeat) comes as a [replacement](https://www.elastic.co/guide/en/beats/filebeat/current/migrating-from-logstash-forwarder.html) to Logstash Forwarder and is based on its source code. Filebeat is a lightweight shipper written in Go that takes very little resources. During my testing, it took about 10-20 Mb of memory which is nothing.

I want to give a quick overview of how Filebeat works and its main features. Of course, you can read [documentation](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-overview.html) for more information.

To put it simply, Filebeat reads the log lines from the files you specify in the configuration and forwards the log data to the configured output. A sample configuration file for Filebeat may look like this:

~~~yml
name: "test-shipper" # if  empty, the hostname is used
tags: ["web-tier", "web-application"]

filebeat.prospectors:
- paths:
    - /var/log/apache2/access.log
  fields:
    apache-access: true
  tags: ["apache"]
#  exclude_lines: ['HTTP']  #  drop any lines that contain  "HTTP".

- paths:
    - /var/log/*.log
    - /var/log/syslog
  scan_frequency: 2 #  how often the prospector checks for new files in the specified paths
  document_type: syslog # event type to use for published lines read by harvesters.

output.logstash:
  hosts: ["localhost:5044"]
~~~

As you can see, in the [general configuration](https://www.elastic.co/guide/en/beats/filebeat/current/configuration-general.html) we define shipper's name and a list of tags ```web-tier, web-application``` which the Filebeat adds to each published event. This could be useful for grouping servers. Imagine that you have a cluster of web servers, then you can add a tag ```web-tier``` to the Filebeat configuration on each server and use filters and queries in the web interface connected to the storage (like Kibana) to get the information about our group of servers.

If you want to tag some specific events, we can use ```tags``` option inside a prospector block as we do for Apache access log. Again, such tags could make the search easier in the web interface and be useful when filtering log traffic in indexer.

Filebeat consists of 2 main components: [prospectors and harvesters](https://www.elastic.co/guide/en/beats/filebeat/current/how-filebeat-works.html). A _harvester_ is responsible for reading lines from a _single_ file and sending them to the output. One harvester is started for each file. A _prospector's_ responsibility is to manage harvesters and find sources to read from.  
You can see that in our config file we used a glob pattern for the log files:
~~~yml
- paths:
    - /var/log/*.log
    - /var/log/syslog
~~~
Prospector's job is to find all the files matching the glob pattern ```*.log``` inside ```/var/log/``` directory and start a harvester for each of them.
We can regulate how often a prospector will look for new files in the paths with ```scan_frequency``` parameter which takes a number in seconds.

In the prospector's block we can also define new fields that we want to be added to each event. These can later be used for filtering.
~~~yml
filebeat.prospectors:
- paths:
    - /var/log/apache2/access.log
  fields:
    apache-access: true
~~~
In general, Filebeat borrows many features from Logstash. It can do log filtering using configuration options like [exclude_lines](https://www.elastic.co/guide/en/beats/filebeat/current/configuration-filebeat-options.html) and [include_lines](https://www.elastic.co/guide/en/beats/filebeat/current/configuration-filebeat-options.html) with a required regular expression. It can also process logs with [processors](https://www.elastic.co/guide/en/beats/filebeat/5.2/configuration-processors.html) you define. The log processors allow to [reduce](https://www.elastic.co/guide/en/beats/filebeat/5.2/drop-fields.html) the number of exported fiels, [add metadata](https://www.elastic.co/guide/en/beats/filebeat/5.2/add-cloud-metadata.html), and perform [decoding](https://www.elastic.co/guide/en/beats/filebeat/5.2/drop-fields.html).

To define a processor, we specify a processor's name, a condition (if not present, it will always be applied), and a set of processor's parameters.
~~~yml
processors:
 - <processor_name>:
     when:
        <condition>
     <parameters>
~~~
You can see the full list of processors at the bottom of this [page](https://www.elastic.co/guide/en/beats/filebeat/5.2/configuration-processors.html). Note that there are not many of them currently available (only five at the time of writing), just enough to keep filebeat a lightweight log shipper and yet give us options to change the log data on the _host side_, so we could make the log processing easier on the _indexer side_.

Another important thing to note is the ability of the Fileabeat to handle [multiline log messages](https://www.elastic.co/guide/en/beats/filebeat/current/multiline-examples.html). This is one of the main concerns when, for example, you need to collect _java stack traces_. Filebeat seems to deal with this problem very well. It has the same functionality as a [multiline codec](https://www.elastic.co/guide/en/logstash/current/plugins-codecs-multiline.html) for Logstash. You'll basically need to define a regular expression (```pattern```) to match, specify how to combine mathing lines with ```match``` option, and you can also set a ```negate``` option to negate the pattern.

On this [page](https://www.elastic.co/guide/en/beats/filebeat/current/multiline-examples.html#_line_continuations), you'll find a link to the [Go Playground](https://play.golang.org/p/uAd5XHxscu) where you can try different regular expressions for your multiline log messages until you get it right.

And there's one more thing I want to mention about filebeat before we wrap it up. It is well known that for stable performance under high load Logstash indexer needs a _message broker_, because Logstash cannot buffer itself unless it reads from a file. So they also made the Filebeat capable of sending logs to [Kafka](https://www.elastic.co/guide/en/beats/filebeat/current/kafka-output.html) and [Redis](https://www.elastic.co/guide/en/beats/filebeat/current/redis-output.html).

#### Conclusion:
We talked about Logstash as a shipper in the beginning of this post and continue to explore it later when we'll be looking at _logs indexers_.

As a shipper, Logstash is too resource-consuming and slow, so we wouldn't want to install it on each of our hosts. Nevertheless, Logstash is a very flexible tool with lots of plugins which allow you to do almost anything with your logs. That's why it's widely used as a log indexer. We'll talk more about it when we'll be exploring the tools at a log indexer level.

Logstash Forwarder and its successor Filebeat were introduced to fill in the need for a lightweight shipper.

Filebeat is a great tool, still young and yet already very powerfull. It takes fewer resources than Logstash and has many of the same features. It integrates perfectly with Logstash and Elasticseach and is capable of writing log data into brokers which Logstash needs for better perfomance.

Another huge advantage of the Filebeat is well-written documentation. You can start using Filebeat in no time. The configuration file in YAML format looks clean and easy to manage. Documentation is great and easy to read, you can find information on every possible configuration option many of which come with examples. Elastic also have a [discuss forum](https://discuss.elastic.co/) where you can ask any question about their products and get help with your configuration.

Overall experience of using Filebeat was awesome.

_Let's see what other alternative log shippers we can use in our next post ..._

_**P.S.** You can use my [logging-sandbox](https://github.com/Artemmkin/logging-sandbox) repository to quickly set up a logging system you want and try it yourself._
