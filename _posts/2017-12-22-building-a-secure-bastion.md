---
layout: post
title:  "Building a secure bastion host, or, 50 ways to kill your server"
tags:   [technical, aws, ami, packer, ubuntu]
date:   2017-12-22
comments: true
---

>  Bastion (noun)
>    1. A projecting part of a fortification
>    2. A special purpose computer on a network specifically designed and configured to withstand attacks

If you deploy servers to a private network, then you also need a way to connect to them. The two most common ways methods are to use a VPN, or to ssh through a bastion host (also known as a jump box). Shielding services this way massively reduces your attack surface, but you need to make sure that the server exposed to the internet is as secure as you can make it.

At [Telenor Digital](https://www.telenordigital.com/) we have about 20 federated AWS accounts, and we wanted to avoid having to set up a complex system of VPNs. Additionally, we wanted to be able to connect to any account from anywhere, and not just from designated IP ranges. Deploying a bastion host to each account would allow us to connect easily to instances via ssh forwarding. Our preferred forwarding solution is [sshuttle](https://github.com/apenwarr/sshuttle), a "transparent proxy server / poor man's VPN".

This is where it got... interesting. At the time of writing, Amazon AWS did not have a designated bastion host instance type available. Nor in fact do any of the other main cloud providers, nor did there appear to be any other trustworthy bastions available from other sources. There didn’t even seem to be any information about how other people were solving this problem.

I knew that we wanted a secure bastion such that:

1. Only authorised users can ssh into the bastion
2. The bastion is useless for anything BUT ssh'ing through

How hard could making such a bastion possibly be?

## Constraints and processes

Our technology stack uses Ubuntu exclusively, and we wanted the bastion to be compatible with the various services we already deploy, such as Consul, Ansible, and Filebeat. Beyond that, I personally have a lot more experience with Ubuntu than I do with any other OS.

For these reasons we decided to base the bastion on a minimal Ubuntu install, strip out as many packages as possible, add some extra security, and make a golden image bastion AMI. Had it not been for these constraints, there might be better OSs to start with, such as Alpine Linux.

Additionally, we run everything in AWS so one or two points of the following are AWS-specific, but based on a lot of conversations it seems that the bastion problem is one that affects a much wider range of architectures.

We use [Packer](https://www.packer.io/) to build our AMIs, [Ansible](https://www.ansible.com/) to set them up and [Serverspec](http://serverspec.org/) to test them, so building AMIs is a pretty fast process, typically taking about five minutes. After that we deploy everything using [Terraform](https://www.terraform.io/), so it's a quick turnaround from code commit to running instance.

## Starting point: Ubuntu minimal-server

My first port of call: what packages are pre-installed in an Ubuntu minimal-server? Inspection via `$apt list --installed` or `$ dpkg-query -W` showed over 2000 packages, and of those I was surprised how many I'd never heard of. And of the ones I had heard of, I was further surprised how many seemed, well, superfluous.

I spent some time and made a few spreadsheets trying to figure out what all the mystery packages were before I got bored and had the bright idea of leveraging Ubuntu's package rating system: all packages are labelled as one of: required, important, standard, optional, or extra.

```
$ dpkg-query -Wf '${Package;-40}${Priority}\n'
apt                             important
adduser                         required
at                              standard
a11y-profile-manager-indicator  optional
adium-theme-ubuntu              extra
```

## Remove optional and extra packages

Those optional and extra packages sounded very nonessential. I was pretty sure I could rip those out with a nice one-liner and be done.

```
dpkg-query -Wf '${Package;-40}${Priority}\n' | awk '$2 ~ /optional|extra/ { print $1 }' | xargs -I % sudo apt-get -y purge %
```

Turns out this was not my best ever idea.
All sorts of surprising packages were marked optional or extra and were thus unceremoniously removed, including:

* cloud-init
* grub
* linux-base
* openssh-server
* resolvconf
* ubuntu-server (meta-package)

It doesn't take a genius to realise that removing grub, open-ssh or resolvconf is colossally ill-advised, but even after I tried not removing these but uninstalling the rest I had no luck. On every build I got an unstable and/or unusable image, often not booting at all. Interestingly it broke in a different way each time, possibly something to do with how fast it was uprooting various dependencies before it got to an unbootable state. After quite a lot of experimenting with package-removal lists and getting apparently nondeterministic results, it was time for a new strategy.

## Remove a selected list of packages

I revised my plan somewhat in the realisation that maybe blindly removing lots of packages wasn't the best of ideas. Maybe I could look through the package list and remove the ones that seemed the most 'useful' and remove them. Some obvious candidates for removal were the various scripting languages, plus tools like curl and net-tools. I was pretty sure these were just peripherals to a minimal server.

* curl: can't remove due to Consul dependencies
* ed
* ftp
* gawk
* nano
* net-tools: can't remove due to sshuttle dependencies
* perl: can't remove due to ssh dependencies
* python 2.7: can't remove due to Ansible dependencies
* python 3: can't remove due to AWS instance checks dependencies
* rsync
* screen
* tar: can't remove due to Ansible dependencies
* tmux
* vim
* wget

It turns out I was incorrect. Due to the various restrictions placed upon the system because we use Consul, sshuttle, Ansible and AWS, about half of my hitlist was unremovable.

To compensate for the limitations in my "remove all the things" strategy, I decided to explore limiting user powers.

## Restrict user capabilities

Really, I didn't want users to be able to *do* anything - they should only be allowed to ssh tunnel or sshuttle through the bastion. Therefore locking down the specific commands a user could issue ought to limit potential damage. To restrict user capabilities, I found four possible methods:

* Change all user shells to /bin/nologin
* Use rbash instead of bash
* Restrict allowed commands in authorized_keys
* Remove sudo from all users

All seemed like good ideas - but on testing I discovered that the first three options only work for pure ssh tunnelling, and don’t work in conjunction with sshuttle.

## Remove sudo

I disabled sudo by removing all users from the sudo group, which worked perfectly apart from introducing a new dimension to bastion troubleshooting - without sudo it’s not possible to read the logs nor perform any meaningful investigation on the instance.

We offset most of the pain by having the bastion export its logs to our ELG (Elasticsearch, Logstash, Graylog) logging stack, and export metrics to Prometheus. Between these, most issues were easily identified without needing direct sudo access directly. For the couple of bigger build issues that I had in the later stages of development, I built two versions of the bastion at a time, with and without sudo. A little clunky, but only temporarily.

With the bastion locked down as much as possible, I then added in a few more restrictions to finalise the hardening.

## Install fail2ban

An oldie but a goodie, fail2ban is fantastic at restricting logon attempts by locking out anyone who fails to login three times in a row for a determined time period.

## Use 2FA and port knocking

Some clever folks in my team ended up making a version of sshuttle that invokes AWS two-factor authentication for our users, and implements a port knocking capability which only opens the ssh port in response to a certain request. The details of this are outside the scope of this article, but we hope to make this open-source in the near future.

## Finally! A bastion!

After a lot of experiments and some heavy testing, the bastion was declared production-ready and deployed to all accounts. The final image is t2.nano sized, but we’ve not seen any problems with performance so far as the ssh forwarding is so lightweight.

It's now been in use for a least half a year, and it's been surprisingly stable. We have a Jenkins job that builds and tests the bastion AMI on any change to the source AMI or on code merge, and we redeploy our bastions every few weeks.

I still lie awake in bed sometimes and try to work out how to build a bastion from the ground up, but to all intents and purposes I think the one we have is just fine.

## Related content

I presented this work at DevOpsDays Oslo 2017, see the video or slides here:

    https://www.youtube.com/watch?v=Kd4OMVYTwnYa
    https://www.slideshare.net/AnnaKennedy11/building-a-secure-bastion-or-50-ways-to-kill-your-server-81551575

This post was first published at Sysadvent 2017:

    http://sysadvent.blogspot.no/2017/12/day-22-building-secure-bastion-host-or.html

## References

    https://www.cyberciti.biz/faq/linux-bastion-host
    https://askubuntu.com/questions/79665/keep-only-essential-packages
    http://docs.aws.amazon.com/quickstart/latest/linux-bastion/architecture.html
    http://annaken.github.io/testing-packer-builds-with-serverspec


