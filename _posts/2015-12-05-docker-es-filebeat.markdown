---
layout:     post
title:      "Docker, Filebeat, Elasticsearch, and Kibana"
subtitle:   "and how to visualize your container logs"
date:       2015-12-05 15:25:23
author:     "Erwin Embsen"
header-img: "img/post-bg-01.jpg"
---

<p>Yesterday, I was looking for a way to view container logs in Kibana. I know how to do that in a VM environment,
however, doing the same thing in a Docker environment was a bit of a challenge to me. I wanted an all Docker solution
such that no software installations outside of Docker were needed to get things up and running. Here is the story.</p>

<h2 class="section-heading">The past</h2>

<p>Not so long ago, I used logstash forwarder to ship application logs to logstash "processors" which in turn streamed
their output into Elasticsearch. Very recently, logstash forwarder has been replaced by Filebeat, part of the new
<a href="https://www.elastic.co/blog/beats-1-0-0">Beats platform</a>. The chaining of the application, filebeat,
logstash and elasticsearch could be as straightforward as shown below:<p>

<a href="#">
    <img src="{{ site.baseurl }}/img/docker-filebeat-elasticsearch.jpg" alt="Chaining application, filebeat, logstash and elasticsearch">
</a>

<p>The arrows show how application log data flows through various components.</p>

<h2 class="section-heading">The present</h2>

<p>On a Unix system, Docker containers usually log their output in log files in directory
<code>/var/lib/docker/containers</code>. The following piece of configuration tells Filebeat to monitor these log
files and collect the logs:<p>

<pre>
filebeat:
  # List of prospectors to fetch data.
  prospectors:
    -
      paths:
        - /var/lib/docker/containers/*/*.log
      input_type: log
      ignore_older: 5m
  registry_file: /var/lib/filebeat/registry
</pre>


<p>Filebeat has an elasticsearch output provider so Filebeat can stream output directly to elasticsearch. That saves us
the logstash component. All you need is this little piece of code in the filebeat.yml configuration file:</p>

<pre>
output:
  elasticsearch:
    hosts: ["es.docker:9200"]
</pre>

<h4 class="section-heading">Let's put the pieces together</h4>

<p>So what we need is a set of 3 containers, "filebeat", elasticsearch, and kibana. And while new is always better, why
not download the latest official elasticsearch and kibana images straight from Docker Hub. The Docker file for the
filebeat container looks like this:</p>

<pre>
FROM debian:jessie

ENV FILEBEAT_VERSION 1.0.0
ENV FILEBEAT_SHA1 824c0b3dce16e3efd7b72b5799e97cc865951ade

RUN apt-get -y update
RUN apt-get -y install apt-transport-https
RUN apt-get -y install curl
RUN curl https://packages.elasticsearch.org/GPG-KEY-elasticsearch | apt-key add -
RUN echo "deb https://packages.elastic.co/beats/apt stable main" | tee -a /etc/apt/sources.list.d/beats.list

RUN apt-get -y update
RUN apt-get -y install filebeat

ADD filebeat.yml /etc/filebeat/filebeat.yml

ENTRYPOINT ["/usr/bin/filebeat", "-e", "-v", "-c", "/etc/filebeat/filebeat.yml"]
</pre>

<p>If you want to build the Filebeat container yourself you can check out my <a href="https://github.com/eembsen/docker-filebeat">GitHub project</a>.<p>

<p>Ok, we are all set. Let's fire up the 3 containers using the following commands:<p>

<pre>
docker run --detach --name=elasticsearch elasticsearch:2.1
docker run --detach --name=kibana --link elasticsearch:elasticsearch -p 5601:5601 kibana:4.3
docker run --detach --name=docker-filebeat -v /var/lib/docker:/var/lib/docker --link elasticsearch:es.docker docker-filebeat
</pre>

<p>Now point your browser at your docker host port 5601 and view the container logs!<p>

<h2 class="section-heading">The future</h2>

<p>This solution leaves a couple of things to be desired. Probably the most important thing is that Docker log files are
accessed directly by Filebeat through a bind mount. If Docker decides to alter their implementation then it might break
the recipe presented here.</p>

<p>Also, containers will not show up in Kibana by their name but by id. It would be nice to have a name instead of an
id. Furthermore, Filebeat currently lacks multiline support. It's on the agenda and filed under
<a href="https://github.com/elastic/filebeat/issues/301">issue 301</a> so I guess we have to wait a little.<p>

<p>Currently, Filebeat either reads log files line by line or reads standard input. A JSON prospector would safe us a
logstash component and processing, if we just want a quick and simple setup.</p>

<p>Photographs by <a href="https://www.flickr.com/photos/nasacommons/">NASA on The Commons</a>.</p>
