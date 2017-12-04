# Building a secure bastion host

>  Bastion (noun)
>    1. A projecting part of a fortification
>    2. A special purpose computer on a network specifically designed and configured to withstand attacks

![Bastion](/assets/img/bastion.jpg)

If you deploy servers to a private network, then you also need a way to connect to them. The two most common ways methods are to use a VPN, or to ssh through a bastion host (also known as a jump box). Shielding services this massively reduces your attack surface, but you need to make sure that the server exposed to the internet is as secure as you can make it.

Here at Telenor Digital we use multiple federated AWS accounts, and we wanted to avoid having to set up a complex system of VPNs. Additionally, we wanted to be able to connect to any account from anywhere, not just from designated ip ranges. Deploying a bastion host to each account would allow us to connect easily to instances via [sshuttle](https://github.com/apenwarr/sshuttle).

This is where it got... interesting. We use Amazon AWS, and at time of writing there was no designated bastion host instance type available. Nor could I find any pre-built bastions. Or even any of information about how other people were solving this problem.
I knew that I wanted a secure bastion such that:

1. Only authorised users can ssh into the bastion
2. The bastion is useless for anything BUT ssh'ing through

And so under the banner of "I'll build it myself, how hard can it be" I set sail into uncharted territory.

## Disclaimer: constraint and pre-conditions

Our technology stack uses exclusively Ubuntu, and we wanted the bastion to be compatible with the various services we already deploy, such as Consul, Ansible, and Filebeat. Additionally, I personally have a lot more experience with Ubuntu than I do with any other OS.
Thus the decision was taken to base the bastion on a minimal Ubuntu install, strip out as many packages as possible, add some extra security, and make a golden image bastion AMI.
Had it not been for these constraints, there might be better OSs to start with, such as Alpine Linux.

## The starting point: Ubuntu minimal-server

First point of call: what packages are pre-installed in an Ubuntu minimal-server?
Inspection via `$apt list --installed` or `$ dpkg-query -W` showed over 2000 packages, and of those I was surprised how many I'd never heard of.
Of the ones I had heard of, I was further surprised how many seemed, well, superfluous.
I spent half a day trying to figure out what all the mystery packages were before I had the bright idea of leveraging Ubuntu's package rating system: all packages are labelled as one of: important, required, standard, optional, or extra

```
$ dpkg-query -Wf '${Package;-40}${Priority}\n'
apt                             important
adduser                         required
at                              standard
a11y-profile-manager-indicator  optional
adium-theme-ubuntu              extra
```
# Remove optional and extra packages

Those optional and extra packages sounded very non-essential. I was pretty sure I can rip those out with a nice one-liner and be done.

```
dpkg-query -Wf '${Package;-40}${Priority}\n' | awk '$2 ~ /optional|extra/ { print $1 }' | xargs -I % sudo apt-get -y purge %
```

Turns out this was not my best ever idea.
All sorts of surprising packages are marked optional/extra and were thus unceremoniously removed, including:

* cloud-init
* grub
* linux-base
* openssh-server
* resolvconf
* ubuntu-server (meta-package)

It doesn't take a genius to realise that removing grub, open-ssh or resolvconf is colossally ill-advised, but even after I tried not removing these but uninstalling the rest I had no luck.
On every run I got an unstable and/or unusable image, often not booting at all.
Interestingly it broke in a different way each time, possibly something to do with how fast it was uprooting various dependencies before it got to an unbootable state.
After a good day of experimenting with package-removal lists and getting apparently non-deterministic results, it was time for a new strategy.

# Remove a selected list of packages

I revised my plan somewhat in the realisation that maybe removing lots of packages blindly isn't the best of ideas.
Maybe I could look through the package list and remove the ones that look most 'useful' and remove them.

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




