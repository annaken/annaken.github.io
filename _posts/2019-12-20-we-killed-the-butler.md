---
layout: post
title:  "We killed the butler: Replacing Jenkins with Concourse"
tags:   [technical, cicd, jenkins, concourse]
date:   2019-12-20
comments: true
---

At Working Group Two, we try to use CI/CD pipelines to automate all of our repetitive tasks when it comes to code and infrastructure deployment and testing, such as:

running unit tests on each pull request
building and running integration tests with bazel on every merge to the monorepo
building container images and upload them to the registry
scanning all images for security flaws
running acceptance tests in the staging environment
syncing secrets between different sources
notifying slack if changes are made in Kubernetes

We had been using Jenkins to run such pipelines, but decided to replace it with Concourse. [Read the rest of the blog here](https://wgtwo.com/blog/replacing-jenkins-with-concourse/).

