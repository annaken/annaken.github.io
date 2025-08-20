---
layout: post
title:  "From infra with love"
tags:   [technical, cicd, gitops]
date:   2025-05-08
comments: true
---

This post was first presented as a talk at [Data Innovation Summit 2025](https://datainnovationsummit.com/).

---

What I’d like to talk to you about today is what the new and exciting world of data platform engineering can steal/learn from the world of infrastructure engineering (spoiler: quite a lot).

To introduce myself a bit more and give you some context for this presentation, my background is originally physics, but I moved into the infrastructure space in about 2010, right as the DevOps movement exploded.

I started off working in ops, then became a sysadmin, then an infrastructure engineer, then an SRE, then a cloud engineer, then a data platform engineer. Despite all the different names, the essential core of all these jobs is the same: deploy and maintain common infrastructure so that other people can focus on their specialisation.

And at the same time, the WAY that I have done this job has changed enormously, and seemingly continuously, over the last 15 years. I could probably talk for hours about all the details and nuance in this ever changing landscape, but I only have twenty minutes, so I’m going to give you my top three principles, and the two biggest red flags that let me know where we have work to do.

![nginx-printout](/assets/images/nginx-analog.jpg)

This is a photo from 2012, when our load balancer config had grown organically until it was impossible to tell what was happening or who owned which section. In the absence of git, we figured the easiest way to fix it was to print out the entire thing (30m long) and have the developers come over and annotate or cross out the sections that they knew about.

It worked remarkably well actually, but I’m thrilled that we don’t have to do it like that any more.

Once the print out was all annotated, we were ready to update the code. At this time, we only had one production load balancer, so until this point we just logged into it, edited the config, and restarted the process. Terrifying. So as part of this updating project, we first put this config into version control so that if we fat fingered something, we could revert to a working version. Which brings us to the first principle:

## Principal 1: everything as code - everything in a repo

Having everything you care about written as code and saved to version control is the foundation upon which everything else is going to be built.

And by everything I do mean everything.

Your infrastructure wants to be saved as code, you could use terraform or cloudformation or Puppet, but you definitely want to be able to deploy and redeploy all of your environments programmatically. I can’t emphasise enough how much time, energy and stress you will save here, in my opinion this is the most important thing of all.

Saving configs as code can also save you a lot of tears. Many graphical interfaces have import/export config options, so even if you have to configure them manually the first time, you’ll want to export the resulting config and save that in a repo so that you can restore it in the future.

Documentation can also live happily in version control, and it’s arguably best to store docs as close as possible to the code they describe. Having it in git also makes it easier to see which documents are relatively up to date, and which ones are old and can be deprecated. 

Once everything is in a repo you will have a much happier time when it comes to 

* knowing what is running
* restoring the current state if something goes horribly wrong
* collaborating 
* reviewing changes
* scaling

Having everything in code is also the prerequisite for principal 2:

## Principal 2: Automate all the things

Every predictable, repeatable process can and should be automated. 

And this doesn’t have to be a big-bang thing, most systems are modular enough that you can automate piecemeal and develop the full pipeline slowly and carefully.

Personally I am a huge fan of gitops, that is, when a git commit alone initiates a workflow. 

The second stage of our loadbalancer revamp was to put some automation in place that would take an update to the code in the repo and deploy it to the live server.

Over time this became a much more sophisticated workflow that looked like:

* A user makes a commit to git
* Deploy to dev
* Run unit tests
* Deploy to staging
* Run smoke and integration tests
* Deploy to prod

You could conceivable automate:

* Deployment of infrastructure from terraform
* Ingesting csv files from an upload server into bigquery
* Updating 3rd party libraries to the latest version

Automating your toil is an amazing way to achieve more with less, but with one big caveat:

## Principal 3: With great power comes great responsibility

The downside of automating all the things is that you have the potential to roll out destructive changes in a much more efficient manner than before, so we need to be responsible citizens and take this power seriously.

The first thing we need to get in place is visibility. How do you know your deploy worked? How can you be sure that the latest loadbalancer change didn’t just, say, cut off all users in Mexico?

We need solid monitoring to see the effects, or lack of effects, of all changes as they are rolled out. If something isn’t working we need to know as fast as possible, ideally before it hits prod. 

With good alerting in place, you can relax a bit more and know that even if you’re not following all your graphs on the screen, you will be notified if something goes wrong. 

And if something doesn’t work and needs to be rolled back, it’s best to prepare a rollback plan in advance. That might look like reverting a commit, falling back to a previous image, or restoring a snapshot. The higher the stakes, the more time you will want to invest in this. 

I guarantee that you don’t want the first time you think about rolling back to be during a production outage.

What we eventually implemented for our loadbalancer was a combination of all of these ideas. 

As a new commit was rolled out through the environments, if the automated testing failed or other alerts were triggered the deploy was halted and automatically reverted to the previous working state. 

Those are my three top principles, and now I want to mention the two biggest signs I look for when I’m thinking about where to focus my efforts - we can’t do everything at once, so what’s the most important thing to start with?

## Flag 1: It shouldn’t be scary

I’m probably most sensitive to this because I have done a lot of scary work, and there’s nothing like the feeling in your stomach when you realise a mistake right after you hit enter. This may or may not have happened with the loadbalancer in the middle of the night once.

So I’m always on the lookout for that feeling - if I find some internal resistance against making a change, restarting a service, updating a config - then that is definitely a thing that needs attention.

I believe strongly in the idea that people will always make mistakes, so you need to design systems to be resilient - ideally it shouldn’t even be possible to have a production outage. 

## Flag 2: Remember that you won’t remember

This is the other thing I’m on the lookout for - I don’t have the best memory, and honestly, I’m a human, our memories are unreliable at best. Compared to a computer, which are on the whole great at remembering things.

If I ever come across a memory step in a process - like remembering to add a label, or kick off a build - that’s the other biggest indicator of something worth putting some effort in to fix. 

People don’t remember, people are sometimes off sick, or change jobs, and when you realise you haven’t ingested any new data since Bob left you’re going to wish you'd automated more of your process.


