# Building a secure bastion host

>  Bastion (noun)
>    1. A projecting part of a fortification
>    2. A special purpose computer on a network specifically designed and configured to withstand attacks

If you deploy servers to a private network, then you need to connect to them either through a VPN or by ssh'ing through a bastion host (jump box). This massively reduces your attack surface - but you need to make sure that the server exposed to the internet is as secure as you can make it.

![Bastion](/assets/img/bastion.jpg)

My version of secure is:

1. Only authorised users can ssh into the bastion
2. The bastion is useless for anything BUT ssh'ing through

Disclaimer: the technology stack we use here at Telenor Digital influenced some design decisions here. Namely, we run everything in AWS, and we use Ubuntu everywhere. Additionally, we need the bastion to be able to run filebeat, Consul, Ansible and sshuttle. Therefore I didn't try to build an image from scratch, nor was it practical for me at this time to try to use a non-Ubuntu distro.

My rough plan was: start with a minimal Ubuntu server AMI, strip out as many packages as possible, add some extra security, and make a golden image bastion AMI.

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




