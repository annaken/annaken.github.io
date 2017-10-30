---
layout: post
title:  ""
tags:   [technical, logstash, s3]
date:   2017-07-18
comments: true
---

In the setup we use at work, we archive logs from Logstash to s3 using the [logstash s3 output plugin](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-s3.html). In the older versions of Logstash (5.1 and before), we implemented the patch described [here](http://www.tothenew.com/blog/tweaking-logstashs-s3-plugin-to-create-folders-in-yyyymmdd-format-on-aws-s3/).
