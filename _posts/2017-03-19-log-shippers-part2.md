---
layout: post
title: >
  Building a centralized logging system. <br /> Log shippers (part 2).
tags: [logging, logship, bigd]
---
### Fluentd
We continue to talk about _log shippers_ and in this part we're going to look at [Fluentd](http://www.fluentd.org/)  which is next on our [list](/2017/03/13/log-shippers-part1/).

Fluentd is another popular tool which is used as a _log shipper_ and _log aggregator_. People often compare it to Logstash and use it as an alternative. So you might find another version of a popular _ELK_ (Elasticseach+Logstash+Kibana) stack which is called _EFK_ (Elasticsearch+Fluentd+Kibana).

![300x300](/public/img/logging/fluentd2.png)  

Nowadays, you'll also see a lot of people use Fluentd for _container logging_. In fact, it was adapted as a main log collector in Kubernetes. All in all, Fluentd seems very interesting, so let's find out what it is like ...<!--break-->

In this post, we will give a quick overview of the Fluentd's main features and mention its strengths and weaknesses.

Guys from Treasure Data who developed Fluentd say that Fluentd was built with the purpose of solving the problem of a logging mess, when you have a whole lot of scripts which collect data from various services, possibly do some parsing and transformation and output it to a bunch of different backend systems. As the result, your logging system looked like this:

![400x400](/public/img/logging/fluentd-before.png)

So they introduced this idea of a [Unified Logging Layer](http://www.fluentd.org/blog/unified-logging-layer) whose goal was to connect various sources of log data to different destination systems (NoSQL databases, HDFS, RDBMs, etc.) and have all the routing, filtering, and aggregation in one single piece of middleware. This naturally decreases the level of complexity of your logging system.

![400x400](/public/img/logging/fluentd-architecture.png)

Although this idea doesn't seem new now and popular log aggregators like _Logstash_ serve the same purpose, we must say that Fluentd was a good attempt at creating this Unified Logging Layer. It has a pluggable architecture with more than 200 opensource community supported output plugins. So basically, there's a plugin for every destination you want. Variety of plugins definitely makes Fluentd a very versatile tool and a strong opponent to Logstash.

Another idea as part of the Unified Logging Layer was to use a standard format for logs. So Fluentd tries to structure data as JSON as much as possible. The downstream data processing becomes much easier with JSON.


### Let's see how Fluentd works.

![400x400](/public/img/logging/fluent-int-architecture.jpg)

Here is the basic flow.
#### Input
When a log message comes in, it is assigned a ```timestamp``` and a ```tag```. The message itself is structured as JSON and called a ```record```. The combination of these three is called an event.

Unlike, Filebeat that can only read from files, Fluentd has a lot of options here. Fluentd’s standard input plugins include [http](http://docs.fluentd.org/v0.12/articles/in_http) and [forward](http://docs.fluentd.org/v0.12/articles/in_forward). Http plugin turns Fluentd into an HTTP endpoint to accept incoming HTTP messages whereas forward turns Fluentd into a TCP endpoint to accept TCP packets. Among others, there is a plugin for popular [syslog](http://docs.fluentd.org/v0.12/articles/in_syslog)  and interestingly looking [exec plugin](http://docs.fluentd.org/v0.12/articles/in_exec) which allows you to execute a script and register the output as a record. You can find some of the commonly used input plugins listed [here](http://www.fluentd.org/datasources).

#### Parser
In the same input section you can specify a log parser to use.

*Log parsers let you break a log message into smaller chunks by following a set of rules which you define, so that it can be more easily interpreted and managed in the future.*

You can choose one from the [list of built-in parsers](http://docs.fluentd.org/v0.12/articles/parser-plugin-overview).  For example, Fluentd has predefined parsers for such popular formats as```apache```, ```nginx```, ```syslog```, ```json```. Other commonly used parsers include ```multiline``` which lets you parse multiline logs and ```regexp``` that allows you to write your own regexp to use for parsing. I also liked the fact that, as was in the case of filebeat, they provide a way to test your regexp via [fluentd-ui’s in_tail editor](http://docs.fluentd.org/v0.12/articles/fluentd-ui#intail-setting) or [Fluentular app](http://fluentular.herokuapp.com/).

And you can always [write your own plugin](http://docs.fluentd.org/v0.12/articles/plugin-development#filter-plugins) no matter whether it is for parsing, filtering, or something else. Fluentd supports 6 types of plugins:  [Input](http://docs.fluentd.org/v0.12/articles/input-plugin-overview), [Parser](http://docs.fluentd.org/v0.12/articles/parser-plugin-overview), [Filter](http://docs.fluentd.org/v0.12/articles/filter-plugin-overview), [Output](http://docs.fluentd.org/v0.12/articles/output-plugin-overview), [Formatter](http://docs.fluentd.org/v0.12/articles/formatter-plugin-overview) and [Buffer](http://docs.fluentd.org/v0.12/articles/buffer-plugin-overview).

Let's create a sample input section. Suppose we would like to collect apache access log. Then our input section could look like this:
~~~yml
# input source
<source>
  @type tail # specify which input plugin to use
  path /var/log/apache2/access.log # path to the file to tail
  pos_file /var/log/td-agent/apache-access.log.pos # where to store the last read position
  tag apache.access # tag an event
  format apache2 # built-in parser for apache logs
</source>
~~~
As we can see here, every input section starts with a ```source``` directive. Then we specify a type of input plugin we want to use and its required parameters. In the case, the [in_tail](http://docs.fluentd.org/v0.12/articles/in_tail#format-required) input plugin requires us to specify  ```path``` to the file we want to tail, ```pos_file``` path where to store information about the last read position (used when td-agent restarts), ```tag``` to add to events of this sort, and a ```format```(parser).

Now, an apache log message which generally looks like this:
![200x200](/public/img/logging/fluentd-raw-apache.jpg)
will receive the following form after going through our input section:
![200x200](/public/img/logging/fluentd-event.jpg)

Note that **tagging** is one of the key concepts in Fluentd's work. As we will soon, all the filtering and routing decisions are made based on a tag match.  
#### Filter
After the input section, you can define a set of [filters](http://docs.fluentd.org/v0.12/articles/filter-plugin-overview) you want to apply to your log messages. Filters allow you to filter out events or perform mutation of the incoming data (enrich events by adding new fields, delete or mask fields for privacy).

Let's add a filter for our apache logs. We use a built-in [record_transformer](http://docs.fluentd.org/v0.12/articles/filter_record_transformer#renewrecord-optional) filter plugin to add a new field to our event.
~~~yml
# filter messages with the tag "apache.access"
<filter apache.access>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
  </record>
</filter>
~~~
Here we used Ruby's ```Socket.gethostname``` function to get the hostname.

Our event will now look like this:
![200x200](/public/img/logging/fluent-add-field.jpg)

#### Buffer

Fluentd has a built-in reliability. It supports in memory and file buffering. Buffer plugins are used by buffered output plugins, such as out_file, out_forward, etc.

#### Output
As was mentioned before, Fluentd was destined to solve the problem of different inputs and outputs. So it has an input plugin and output plugin for almost every possible source and destination. Some of the popular output plugins are listed on [this page](http://www.fluentd.org/dataoutputs).

Output plugins are divided into different types based on their buffering implementation. About output pluging types and sorts of buffering you can read [here](http://docs.fluentd.org/v0.12/articles/output-plugin-overview).

Among  _non-buffered_ output plugins which immediately write out results, the most commonly used built-in plugins include [copy](http://docs.fluentd.org/v0.12/articles/out_copy) which allows you to write the same data to multiple outputs and [stdout](http://docs.fluentd.org/v0.12/articles/out_stdout) used for debugging purposes.

We'll use [out_forward](http://docs.fluentd.org/v0.12/articles/out_forward) output plugin in our example to send apache logs to another Fluentd which will act as an aggregator. (I used efk and log-generator playbooks from [this repo](https://github.com/Artemmkin/logging-sandbox)):
1. Output section for _Fluentd shipper_:
    ~~~yml
# output section for logs matching apache.access tag
      <match apache.access>
        @type forward
        send_timeout 1s #  timeout time when sending event logs
        recover_wait 10s #  wait time before accepting a server fault recovery
        heartbeat_interval 1s
        hard_timeout 60s # used to detect server failure
      # where to forward logs
        <server>
          name EFK # name of a fluentd instance to send logs to
          host 172.X.X.X # ip address of a fluentd instance to send logs to
          port 24224
        </server>
      </match>
    ~~~
2. Config for _Fluentd aggregator_.
    ~~~yml
    <source>
      @type forward
      port 24224
    </source>

    <match apache.access>
      @type elasticsearch
      logstash_format true #  make writing data into ElasticSearch indices compatible to what Logstash calls them
    </match>
    ~~~


Now if I search for ```logstash-*``` index in Kibana, I'll be able to see access logs from my apache server.

![400x400](/public/img/logging/kibana-fluentd.jpg)

#### Formatter

For some output plugins like [out_file](http://docs.fluentd.org/v0.12/articles/out_file) that support Text Formatter, you can use  the format parameter to [change the output format](http://docs.fluentd.org/v0.12/articles/formatter-plugin-overview).

### How can I use Fluentd for my application logging?

Collecting logs from various services is great, but we primarily care about the work of our application. So how can we use Fluentd to collect logs from our application?

There are Fluentd [libraries](http://docs.fluentd.org/v0.12/categories/logging-from-apps) for popular programming languages that you can use to post your application logs to Fluentd.
On their website, there are [examples]((http://docs.fluentd.org/v0.12/categories/logging-from-apps)) showing you how simply it is to use this libraries inside your application code.

I decided to try fluentd library for [python](http://docs.fluentd.org/v0.12/articles/python). I took the code they suggest for testing.
```
from fluent import sender
from fluent import event
sender.setup('pythonapp.test', host='localhost', port=24224)
event.Event('follow', {
  'from': 'userA',
  'to':   'userB'
})
```
Then I changed my configuration a little for fluentd instances (all config files you can find in this [repo](https://github.com/Artemmkin/logging-sandbox)) and ran the code.

![400x400](/public/img/logging/python-app.jpg)

It seems to work fine, and considering the fact that forward output plugin provides buffering, it's worth taking a look at fluentd as a way to transport your application logs to a centralized location.

#### Other things to mention

Fluentd package comes with a [web ui](http://docs.fluentd.org/v0.12/articles/fluentd-uihttp://docs.fluentd.org/v0.12/articles/fluentd-ui) which allow you to manage  Fluentd's configuration through the web interface and not only from the command line.

Fluentd is written in Ruby with performance sensitive parts written in C. It takes fewer resources than Logstash (~40 Mb).

There's also another project called [Fluent Bit](http://fluentbit.io/). This is like filebeat for Logstash. It is a super lightweight log shipper which is written entirely in C. Hopefully, I'll have the time to write about it in the next posts, as this one is getting too long.

#### Conclusion

In this post, we looked at another popular log shipper and log aggregator. As we saw, Fluentd is a very flexible and versatile tool with plugins for almost every case you need. It was built on the idea of logging in JSON and creating a Unified Logging Layer for your logging system. It will fit you perfectly, if you have many different sources and destinations.

Fluentd also provides libraries for many popular programming languages, so you can easily plug in your application to your logging pipeline.

Fluentd has a built-in reliability in the form of buffers. And all management (routing, filtering) over log streams is based on tags.
