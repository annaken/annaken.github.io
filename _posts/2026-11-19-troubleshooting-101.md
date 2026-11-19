---
layout: post
title:  "System troubleshooting 101"
tags:   [systems]
date:   2026-11-18
comments: false
---

Troubleshooting a complex system means asking yourself (or your customer) a series of questions to try to define the problem as sharply as possible. If you are able to pinpoint the precise conditions under which a system fails, you can usually work out what the fix should be.

# WHAT

## What is the problem, and how does behaviour differ from what you were expecting?

The initial problem spec for all of us usually defaults to “this thing doesn’t work!” which while not being incorrect, is also not especially helpful in troubleshooting further. Deviation from expected behaviour is what we’re really interested in here.

## Is the problem reproducible?

Intermittent problems are very hard to troubleshoot, and the answer here is often to wait and pounce when it happens again, trying to find a correlation with some other variable, perhaps time, load, or some kind of interaction. Check logs and metrics at similar timestamps, starting local and working your way outwards.

# WHEN

## Did this ever work?

If it never worked, then we’re either talking about a brand-new setup, or a very old setup. Either way, it could mean there might be many parts that aren’t configured correctly, so be prepared to go very broad with the troubleshooting and anticipate multiple failure points.

New system troubleshooting is also where it pays off to have a good enough mental model to break the system down into modular, testable components.

If it used to work but now doesn’t, that (usually) means we can narrow the scope considerably.

## When did the behaviour change? Were there any other changes made at the same time?

Systems tend not to break spontaneously, so it’s worth trying to find out if anything else changed at around the same time as the problem started. It’s usually worth digging through all the changes made to the component and the repo around the same time.

Note that some changes cause problems instantly, whereas some move us into an unstable-but-still-running state, where a future event (time period, load, usage) is what actually causes visible issues.

# WHERE

## Is it a problem in all cases?

Do we see problems with 100% of requests, data, interactions etc, or only a small amount? Partial disruption means that we have a working system under some conditions, but not others. Identifying the differences can be very helpful.

## Are there any situations where this does work?

If you can find a parallel situation where the thing does work, then it’s often just playing spot-the-difference between the two systems - can we find a working example in a different environment, region, timezone? With different user activity, load, settings? At a different scale? If there are no problems for some users (eg, admin) then it’s worth investigating authentication and authorisation, however that is configured for your system.

The more constraints you can compare, the more you can home in on the problem. Be aware that there might be multiple variables at play here. Keep a list as you investigate, and cross off things that cannot be causing the problem, although be prepared to revisit your list as you find out more information. 

> "When you have eliminated the impossible, whatever remains, however improbable, must be the truth" _Sherlock Holmes_

