# Building a secure bastion host

>  Bastion (noun)
>    1. A projecting part of a fortification
>    2. A special purpose computer on a network specifically designed and configured to withstand attacks

![Bastion](/assets/img/bastion.jpg)

If you deploy servers to a private network, then you also need a way to connect to them. The two most common ways methods are to use a VPN, or to ssh through a bastion host (also known as a jump box). Shielding services this massively reduces your attack surface, but you need to make sure that the server exposed to the internet is as secure as you can make it.

Here at Telenor Digital we use multiple federated AWS accounts, and we wanted to avoid having to set up a complex system of VPNs. Additionally, we wanted to be able to connect to any account from anywhere, not just from designated ip ranges. Deploying a bastion host to each account would allow us to connect easily to instances via [sshuttle](https://github.com/apenwarr/sshuttle).

This is where it got... interesting. We use Amazon AWS, and at time of writing there was no designated bastion host instance type available. Nor could I find any pre-built bastions. Or even any of information about how other people were solving this problem. I seemed to have stumbled into the land of [xkcd 979](https://xkcd.com/979). And so under the banner of "I'll build it myself, how hard can it be" I set sail into uncharted territory.

## Where do I start

Our technology stack uses exclusively Ubuntu, and we wanted the bastion to be compatible with the various services we already deploy, such as Consul, Ansible, and Filebeat. Additionally, I feel a lot more comfortable messing with Ubuntu than I would do with other OSs. Had it not been for these constraints, there might be better OSs to start with, such as Alpine Linux.

Thus the decision was taken to base the bastion on a minimal Ubuntu install, strip out as many packages as possible, add some extra security, and make a golden image bastion AMI.


## What does secure mean?

My version of secure is:

1. Only authorised users can ssh into the bastion
2. The bastion is useless for anything BUT ssh'ing through


## The starting point: Ubuntu minimal-server

A quick glance at `dpkg-query -W` shows over 2000 packages pre-installed in Ubuntu minimal-server.

```
$ dpkg-query -W
a11y-profile-manager-indicator  0.1.10-0ubuntu3
accountsservice 0.6.40-2ubuntu11.3
acl     2.2.52-3
acpi-support    0.142
acpid   1:2.0.26-1ubuntu2
activity-log-manager    0.9.7-0ubuntu23.16.04.1
adduser 3.113+nmu3ubuntu4
adium-theme-ubuntu      0.3.4-0ubuntu1.1
adwaita-icon-theme      3.18.0-2ubuntu3.1
```

That's really quite a lot, and even though I thought I knew something about Ubuntu, I don't know what most of these packages are. Still, on closer inspection, Ubuntu marks all of the packages as one of: important, required, standard, optional, or extra. Those optional and extra packages sound very non-essential, I'm sure I can rip those out with a nice one-liner.

```
dpkg-query -Wf '${Package;-40}${Priority}\n' | awk '$2 ~ /optional|extra/ { print $1 }' | xargs -I % sudo apt-get -y purge %
```

Yeah... don't do that. All sorts of surprising packages are marked optional/extra, including:

* cloud-init
* grub
* linux-base
* openssh-server
* resolvconf
* ubuntu-server (meta-package)

On every run I got an unstable and/or unusable image. Interestingly it broke in a different way each time.

Ok so maybe removing lots of packages blindly isn't the best of ideas. Maybe I look through the package list and remove the ones that look most 'useful' and remove them.

Package name | Ok to remove?
------------------------------
curl | no - needed for Consul
ed | yes
ftp | yes
gawk | yes
nano | yes
net-tools | needed for sshuttle
perl | no - needed for ssh
python 2.7 | no - needed for Ansible
python 3 | needed for AWS instance checks
rsync | yes
screen | yes
tar | no - needed for Ansible
tmux | yes
vim | yes
wget | yes











  https://www.youtube.com/watch?v=K72X8_KQeys&feature=youtu.be

  https://www.slideshare.net/AnnaKennedy11/building-a-secure-bastion-or-50-ways-to-kill-your-server-81551575




