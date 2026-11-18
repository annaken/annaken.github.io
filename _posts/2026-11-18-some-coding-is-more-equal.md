---
layout: post
title:  "All coding is equal, but some coding is more equal than others"
tags:   [ai]
date:   2026-11-18
comments: false
---

I feel like in the last few months I’ve read a thousand articles about how coding is solved, how AI will make all software engineers redundant, how everyone is vibe-coding a hundred PRs a day that get merged without review.

This has not been my personal experience so far, so I’ve been feeling a bit confused about the gap between what I’m reading and what I’m seeing at work and among my local peer groups. I feel like at least some of the disconnect here is down to treating “coding” as if it was one thing, rather than a uniquely versatile tool with countless use-cases.

We talk about code as if it’s all one thing, but the variation between use-cases is wild. A piece of software might exist as a personal script to show cat photos on your laptop, or to coordinate the transfer of large sums of money between international banks.

Deciding whether a piece of software needs a human in the loop depends on factors including but by no means limited to:

* User - who is using this, and in what context? 
* Complexity - are we making a complex change?
* Novelty - is this boilerplate, or quite innovative?
* Maturity - are we updating an established system, or making something new?
* Understanding - will a human ever need to understand this code?
* Verification - can we test the outcome before, during and after?
* Consequence - what happens if there’s a failure?
* Degradation - what kind of failure modes can we tolerate?
* Rollback - can we revert a given change?

There are for sure projects where vibe-coding straight to main is absolutely no problem, and other projects where it stands to cause serious harm to someone or their bank account.

Right now on my work slack I see:

> Big kudos to our designers who are now building and shipping code with Copilot! From icon updates and UI polish to new prototypes and the grid behaviour work, it's been so cool to watch. So inspiring to see.

and I think that is a perfect use of AI: the problem is very well defined, the results are seen instantly, the risks are minimal and the change can be reverted easily. 

Not appreciating the difference between yes-vibe and no-vibe projects makes it very hard to write guidelines and guardrails, because we’re trying to write one policy to rule them all (this is especially hard in monorepos). A better approach might be to classify and label projects and changes first, before we decide which path to set off on. 

When I’m reading about the people shipping hundreds of unreviewed PRs a day, I’m wondering what exactly is in those PRs. I’m really hoping it’s not updates to my bank’s money transfer systems, or the autopilot on my self-driving car. 

