---
layout: post
title:  "Forbidden lore: hacking DNS routing for k8s"
tags:   [technical, cicd, jenkins, concourse]
date:   2020-12-11
comments: true
---

At WG2 we’re coming close to having everything running in Kubernetes, which means that almost everything we deploy needs to be pulled from a registry. We have run our own local registry for some time now, to host both locally-built images and cached images from Docker Hub.

We recently decided to improve the registry solution by implementing [Harbor](https://goharbor.io/) to scan images for vulnerabilities on upload, and replicating the registry into each of our multiple environments and regions. This would both eliminate Harbor as a single point of failure, and allow each cluster to pull images locally to minimise data transfer costs through the NAT gateway.

The overall workflow would look something like:

* images are built and uploaded to Harbor
* Harbor scans for vulnerabilities and pushes images to a private registry
* this registry is replicated to a read-only registry
* the read-only registry is replicated to all environments and regions
* the Kubernetes cluster in each environment deploys from the local read-only registry

[Read the rest of the blog here](https://www.wgtwo.com/blog/forbidden-lore-hacking-dns-routing-for-k8s/)
