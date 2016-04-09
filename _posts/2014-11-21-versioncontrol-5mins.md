---
layout: post
title:  "Version control: the basics in 5 minutes"
tage:   [technical, git]
date:   2014-11-21
---

_So, everyone keeps talking to me about this version control thing. What's the beef?_

Version control is really just keeping a history of all the changes you made in a set of documents.

Let's imaging you're working on a project right now, and all the code and configs are stored in a directory. The basic version functions, but you want to add more features. You make some changes, and now things are broken.

However, your directory is under version control, so you can compare what the code looks like now with what the code looked like when it last worked, and see what you changed.
If you want you can just revert to a previous version, undoing as many stages as you need.

_Right now I just copy myfile.txt to myfile.bak, that lets me keep as many versions of my document as I like. And we have daily backups, so I won't lose anything._

And what happens when you make the next version? myfile.bak.2?

_Yep._

And in three years' time when you have made 1,463 changes to the 14 files that make up your project and you need to know when you introduced the bug you just found?

_Sometimes I put the date after .bak..._

And how do you communicate these changes and what they mean to other members of your team? Especially if they work remotely?

_Post-it notes?_

They might kill you after post-it note #1,462.

_Ok so tell me a better way._

If you keep your project under version control, then every time you reach a moment where you want to record the exact state you can 'commit' the current state of all the tracked files.
The commit will have a unique identifier, and with a nice comment so you can browse through your history and see the project evolution, and so that your collaborators in a different timezone can see what you changed.

_How do I share this project with them?_

Usually you'd have a copy of the project on a central repository that everyone can access.

Each team member takes a copy, works on it locally, and then uploads their changes back to the central repo when they're ready.

_What happens if two people change the same file at the same time?_

Then you'll get a conflict, which might be automatically resolved if the changes don't overlap, or might need you to help it decide which revisions are kept.

_Sounds complicated._

It doesn't have to be. Projects can get more complicated if there are lots of contributors and lots of interdependent files, but for a simple project - maybe just a document or a couple of scripts in the beginning - it's no more complicated than the revision history feature of many word processing tools.

_How about if I have a working piece of software and I want to try out adding a new feature?_

Then you probably want to branch the project, where the working software continues along the track it was on, and you go off and work on your branch. When the feature is ready, you can merge the two branches together again.

_I hear a lot of chatter about which software I ought to use, but all these three-letter names mean nothing to me._

In the beginning, there was CVS. Then SVN more or less replaced it by making a better version of the concept. Then git turned up and did everything completely differently. Right now, git and SVN have comparable market shares, but git's popularity is accelerating.

SVN (like CVS) uses a client-server model, where the central server houses the 'definitive' repository, and your laptop is merely a client. Every time a developer makes a change they want to save, this must be written back to the central server, and then updated on
each collaborator's computer.

You can use the same workflow in git, but you don't have to. Git is technically a distributed-model system, which means that everywhere the project lives is a fully-formed repository: your laptop, your colleague's pc, the central git server.

_Why would I want my own repository on my laptop?_

Well, say you're working on your own branch of a project and you go away for a few days to a conference, and the internet is so bad you can't reach the central repository. 
If you're using git you can continue to work on your code offline, commit your changes to your local repo, and when you get back to the office next week you can upload everything you've done back to the central storage area.

The flexibile workflow and clever branching and merging features that git offers are some of the reasons it's become so popular. That
 and Github.

_Github?_

You know, where all the cool kids put their open source stuff.

_I'll have to check that out..._

Do. Git has a bit of a reputation for having a steep learning curve, but if you're starting out with a straightforward project of a couple of files and users it's probably no harder than learning SVN. And frankly about a million times better than having folders full of .bak files. 

_The sixty-four-thousand dollar question: will version control stop my blood running cold when I realise I've messed up all the configs and now everything is on fire?_

Yes. 
(Probably.)


