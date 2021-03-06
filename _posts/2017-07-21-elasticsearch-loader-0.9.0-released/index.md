---
layout: post
title: "Elasticsearch Loader 0.9.0 released"
title-short: Elasticsearch Loader 0.9.0
tags: [elasticsearch, kinesis, aws]
author: Ben
category: Releases
permalink: /blog/2017/07/21/elasticsearch-loader-0.9.0-released/
---

We are thrilled to announce [version 0.9.0][release-090] of Elasticsearch Loader, our component
that lets you sink your [Kinesis streams][kinesis] of Snowplow enriched events to
[Elasticsearch][es].

This release adds support for Elasticsearch 5 and other important features
such as the possibility to use SSL when relying on the REST API of Elasticsearch and the ability
to sign requests when using [Amazon Elasticsearch Service][amz-es].

<!--more-->

In this post, we will cover:

1. [Support for Elasticsearch 5](#es5)
2. [Security features](#sec)
3. [Bug fixes and other minor features](#fixes)
4. [Project modernization](#modernization)
5. [Upgrading](#upgrading)
6. [Roadmap](#roadmap)
7. [Contributing](#contributing)

<h2 id="es5">1. Support for Elasticsearch 5</h2>

In version 0.8.0, we used to provide two artifacts: one for Elasticsearch 1.x and one for
Elasticsearch 2.x, both supporting the REST API as well as the Transport API.

By contrast, version 0.9.0 is split into three artifacts:

- [snowplow-elasticsearch-loader-http][http-artifact]: which lets you communicate with any
Elasticsearch cluster using the REST API (since the REST API is compatible with every Elasticsearch version)
- [snowplow-elasticsearch-loader-tcp][tcp-artifact]: which lets you communicate with your
Elasticsearch 5.x cluster using the Transport API
- [snowplow-elasticsearch-loader-tcp2x][tcp2x-artifact]: which lets you communicate with your
Elasticsearch 2.x cluster using the Transport API

You'll notice that support for the Transport API for Elasticsearch 1.x has been dropped in this
release. Maintaining three different artifacts for the Transport API was becoming a major effort.

Furthermore, [as Elasticsearch is planning on phasing out the transport API][transport-article], we
too will slowly be winding down our efforts around the Transport API. For the discussion on removing Transport API support altogether, see [issue 53][i53].

<h2 id="sec">2. Security features</h2>

This release also brings two important features regarding security: SSL support as well as the
ability to sign AWS requests when using Amazon Elasticsearch service. Of course, both of these
features only affect the HTTP API.

<h3 id="ssl">2.1 HTTPS support</h3>

You'll now be able to use TLS when using the HTTP API, with the following configuration parameter:

{% highlight json %}
elasticsearch {
  client {
    ssl: true
  }
}
{% endhighlight %}

HTTPS support was contributed by [Simon Frid][fridiculous], many thanks Simon!

<h3 id="signing">2.2 Signing AWS requests</h3>

This release also adds the ability to sign your HTTP requests to Amazon Elasticsearch Service, per the process described in [the AWS documentation][signing-requests]. To enable this feature, you will have to modify your configuration file as follows:

{% highlight json %}
elasticsearch {
  aws {
    signing: true
  }
}
{% endhighlight %}

<h2 id="fixes">3. Bug fixes</h2>

<h3 id="buffer-size">3.1 Buffer size</h3>

In Elasticsearch Loader, records from Kinesis are buffered before being sent off to Elasticsearch.
Prior to 0.9.0, the situation where a record size was bigger than the size of the whole buffer was
mishandled and would throw an exception. This has now been fixed, thanks to [Adam Gray][acgray].

<h3 id="max-size">3.2 Maximum payload size</h3>

Another issue, only affecting old (1.x) versions of Elasticsearch, was that payloads
were limited in size to 32 kB, meaning that a record exceeding this size wouldn't be accepted by Elasticsearch.

As a result, this rejected record would end up in the bad Kinesis stream where errors are recorded. However, if that same record also exceeded the maximum size of a Kinesis record (1 MB), then it would be silently lost.

We fixed this issue by truncating the payload contained in the error report sent to the bad Kinesis stream.

<h2 id="modernization">4. Project modernization</h2>

As with our [Kinesis S3 0.5.0 release][ks3-rel], we took advantage of this release to overhaul the
Elasticsearch Loader project by:

- Moving to Scala 2.11 ([#39][i39])
- Moving to Java 8 ([#38][i38])
- Updating the Kinesis Client library ([#51][i51])
- A flurry of [other library updates](https://github.com/snowplow/snowplow-elasticsearch-loader/issues?utf8=✓&q=is%3Aissue%20milestone%3A"Version%200.9.0"%20Bump)

We also took this opportunity to move this codebase out of the core `snowplow/snowplow` repo, into a dedicated [snowplow/snowplow-elasticsearch-loader][repo].

<h2 id="upgrading">5. Upgrading</h2>

A couple of changes have been made to how you run the Snowplwo Elasticsearch Loader.

<h3 id="jar">5.1 Non-executable JARs</h3>

From now on, the produced artifacts will be non-executable JAR files. We found that [sbt-assembly][sbt-assembly], the plugin we use to build fat JARs, was producing executable but unfortunately corrupt JAR files.

As a result, you'll now have to launch the loader like so:

{% highlight bash %}
java -jar snowplow-elasticsearch-loader-http-0.9.0.jar --config my.conf
{% endhighlight %}

<h3 id="conf">5.2 Configuration changes</h3>

There have been quite a few changes made to the configuration expected by the Elasticsearch Loader. Please check out the [example configuration file in the repository][conf-file] to make sure your configuration file has the expected parameters.

<h2 id="roadmap">6. Roadmap</h2>

As mentioned earlier in this post, we're planning on phasing out the support for the Transport API - see [issue 53][i53] for more details.

Additionally, we'd like to extend the scope of this project by having the ability to write
non-Snowplow events to Elasticsearch as [plain JSONs][i26] or [Avro records][i28].

We are overhauling Snowplow Mini to use [NSQ][nsq] instead of Unix named pipes internally; as part of this we will need to extend Elasticsearch Loader to be able to read events from NSQ (see [issue 59][i59]).

Finally, we're also keen on being able to read events from [Apache Kafka][kafka] in addition to
Kinesis, perhaps leveraging [the Kafka Connect project for Elasticsearch][kafka-connect-es]; this is
discussed in [issue 18][i18].

<h2 id="contributing">7. Contributing</h2>

You can check out the [repository][repo] and the
[open issues](https://github.com/snowplow/snowplow-elasticsearch-loader/issues?utf8=✓&q=is%3Aissue%20is%3Aopen%20)
if you'd like to get involved!

[release-090]: https://github.com/snowplow/snowplow-elasticsearch-loader/releases/tag/0.9.0
[conf-file]: https://github.com/snowplow/snowplow-elasticsearch-loader/blob/master/examples/config.hocon.sample
[repo]: https://github.com/snowplow/snowplow-elasticsearch-loader

[kinesis]: https://aws.amazon.com/kinesis/streams/
[amz-es]: https://aws.amazon.com/elasticsearch-service/
[es]: https://www.elastic.co/products/elasticsearch
[kafka]: http://kafka.apache.org
[nsq]: http://nsq.io/

[ks3-rel]: /blog/2017/07/07/kinesis-s3-0.5.0-released/

[http-artifact]: http://dl.bintray.com/snowplow/snowplow-generic/snowplow_elasticsearch_loader_http_0.9.0.zip
[tcp-artifact]: http://dl.bintray.com/snowplow/snowplow-generic/snowplow_elasticsearch_loader_tcp_0.9.0.zip
[tcp2x-artifact]: http://dl.bintray.com/snowplow/snowplow-generic/snowplow_elasticsearch_loader_tcp2x_0.9.0.zip

[i18]: https://github.com/snowplow/snowplow-elasticsearch-loader/issues/18
[i26]: https://github.com/snowplow/snowplow-elasticsearch-loader/issues/26
[i28]: https://github.com/snowplow/snowplow-elasticsearch-loader/issues/28
[i38]: https://github.com/snowplow/snowplow-elasticsearch-loader/issues/38
[i39]: https://github.com/snowplow/snowplow-elasticsearch-loader/issues/39
[i51]: https://github.com/snowplow/snowplow-elasticsearch-loader/issues/51
[i53]: https://github.com/snowplow/snowplow-elasticsearch-loader/issues/53
[i59]: https://github.com/snowplow/snowplow-elasticsearch-loader/issues/59

[fridiculous]: https://github.com/fridiculous
[acgray]: https://github.com/acgray

[sbt-assembly]: https://github.com/sbt/sbt-assembly
[kafka-connect-es]: https://github.com/confluentinc/kafka-connect-elasticsearch

[transport-article]: http://www.elastic.co/blog/state-of-the-official-elasticsearch-java-clients
[signing-requests]: http://docs.aws.amazon.com/general/latest/gr/signing_aws_api_requests.html
