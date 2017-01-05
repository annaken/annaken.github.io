---
layout: post
title:  "TIL how to (and how not to) chain logstash instances"
tags:   [technical, logstash]
date:   2017-01-04
comments: true
---

We have a legacy ELK-stack that has been struggling somewhat lately.
We decided to make a new ELG-stack (Graylog replacing Kibana) in parallel, so that we could have both systems running with live data for some time before we flipped the switch.
This meant that we needed to send the logs from the old ELK-stack to the new ELG-stack, specifically from logstash to logstash.

Turns out the options for doing this are somewhat limited.

My first choice would have been http, but I couldn't get the http input/output plugins to align - the output wants to send logs to a url specifying a verb; the input has no such fields.

I also looked at the lumberjack plugin, but that needed ssl certificates setting up and in our closed, temporary system I felt this was a little overkill (ok, I was lazy).

In the end we plumped for the tcp input/output plugins, which works nicely.

The only drawback with this is that sitting in front of the new logstash instances is an ELB, which then needs to do TCP-loadbalancing. Sounds fine but in practice this means sticky sessions for reasons best known to Amazon.
Which in itself should be ok, but NB! sometimes all the sessions end up stickied to just a couple of servers.
