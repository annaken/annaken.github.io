---
layout: post
title:  "Creating a CentOS base box for Vagrant"
tage:   [technical, vagrant, centos]
date:   2014-07-23
---

I've been using Vagrant for a while, but I recently decided to start making my own base boxes for various reasons (curiosity and paranoia mainly). My Debian base box pretty much 'just worked' thanks to some nice instructions here. However my CentOS (6.5) base box proved a little trickier. This how I eventually got it working.


## Make the VM in VirtualBox


In setup wizard:

* Name: whatever takes your fancy
* RAM: we can set this small, say 512MB, and increase it later
* Hard disk: VMDK, dynamically allocated, again start small, say 10GB


In settings: 

* CPUs: give it a couple if you have the resources
* Disable audio and USB
* Network adapter 1 set to NAT, cable connected, 
* Port forward SSH (TCP) from host port 2222 to guest port 22


## Install the operating system

Download the latest iso and blast through the install. I didn't do anything fancy here.

The root password needs to be 'vagrant'
There also needs to be a user called 'vagrant' with the password 'vagrant'


## Log in as root and update the packages

Do a yum update; if the following are not already installed, yum install them:

* sudo
* ssh-server
* curl
* vim (if you are so inclined, then do update-alternatives --config editor)
* ntp

Set sshd and ntp to auto-start on boot.
Do chkconfig <servicename> on


## Give vagrant user full access to all the things

Run visudo (as root), and add this line to give vagrant passwordless sudo:

    vagrant ALL=(ALL) NOPASSWD:ALL

Allow sudo without tty duing vagrant up:

    Defaults !requiretty

NB I read a lot of advice about just commenting this out, but that didn't work for me - I had to explicity set it to not require tty


## Install Guest Additions

I did the initial install using the GUI, as that seems to make it a bit easier to get the Guest Additions to work / mount / run

Install the dependencies first:

    $ yum --enablerepo rpmforge
    $ yum install dkms kernel-devel kernel-headers linux-headers-server
    $ yum groupinstall "Development Tools"

or you can try limiting this to just gcc and make.

In the VM window, select Devices, Install Guest Additions to mount the virtual CD.
Click on the icon and let it autorun (or mount and run if you're not using the GUI).
I got some errors to begin with; these were fixed by doing this patch for 6.5

    $ cd /usr/src/kernels/<kernel_release>/include/drm/
    $ ln -s /usr/include/drm/drm.h drm.h
    $ ln -s /usr/include/drm/drm_sarea.h drm_sarea.h
    $ ln -s /usr/include/drm/drm_mode.h drm_mode.h
    $ ln -s /usr/include/drm/drm_fourcc.h drm_fourcc.h

 
## Set up ssh access for the vagrant user

Download the standard vagrant public key

    $ wget --no-check-certificate https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub
Save it to .ssh/authorized_keys and make sure the appropriate permissions are set:

* 700 for .ssh
* 600 for .ssh/authorized_keys

And that the whole lot is owned by vagrant:vagrant


## Disable the GUI

In /etc/inittab, set run level to '3' (GUI is '5')


## Networking

Right now my VM only has loopback and eth0 interfaces; this is fine. I want to set eth0 to get assigned an IP by DHCP, and I'll leave it as that for the base box. When I bring up a new vagrant instance I want to be able to assign a static IP on eth1.

This is where I started to run into difficulties. No matter what I set in my ifcfg-eth0/eth1 files, it was completely ignored, and I was unable to stop the VM bringing up eth1 as DHCP and auto-assigning an IP from the VirtualBox DHCP range (192.168.56.101-254, since you ask).

I could probably have coped with the automatically assigned ip address, but it was a matter of principle, plus it annoyed me to have some of my VMs on different subnets.

What it turned out to be was, since I'd installed the OS using a GUI, CentOS had automagically set NetworkManager to 'on' and 'network' to 'off'. (chkconfig --list | grep etwork). This has the effect of disregarding anything set in /etc/sysconfig/network-scripts/.

To get things back to the way I like them, I did:

    $ service NetworkManager stop
    $ chkconfig NetworkManager off
    $ service network on
    $ chkconfig network on

    /etc/sysconfig/network-scripts/
      DEVICE=eth0
      TYPE=Ethernet
      ONBOOT=yes
      NM_CONTROLLED=no
      BOOTPROTO=dhcp

And just to be on the safe side (I was sort of bored with fighting with networks by this point)

    rm etc/udev/rules.d/70-persistent-*

    /etc/selinux/config 
      SELINUX=permissive
      chkconfig iptables off
      chkconfig ip6tables off


## Check stuff

* Reboot
* Check things are on / off / persistent / running as appropriate
* Delete history if you like things tidy
* Shutdown the machine.


## Package as a Vagrant box

Within the folder containing all the vbox files etc, package up your new VM as a Vagrant box:

    $ vagrant package --base centos_6_64 --output centos6.box
    $ vagrant box add centos6 centos6.box


## Make a test Vagrant VM

Now let's make a test Vagrant VM to ensure everything is working ok:

    $ vagrant init

    Vagrantfile:
      # vi: set ft=ruby :
      VAGRANTFILE_API_VERSION = "2"
      Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
              config.vm.box = "centos6"
              config.vm.network "private_network", ip: "192.168.120.40"
              config.vm.hostname = "cirque-centos.vm"
      end

    $ vagrant up
    $ vagrant ssh

All things being well, you should now be able to ssh into the box, and check things look ok.
In particular, check the output of ifconfig and be sure that lo, eth0 and eth1 are configured as required, and that the shared folder mounts correctly under /vagrant.


### References

http://williamwalker.me/blog/creating-a-custom-vagrant-box.html

https://docs.vagrantup.com/v2/boxes/base.html

http://wiki.centos.org/HowTos/Virtualization/VirtualBox/CentOSguest

http://thornelabs.net/2013/11/11/create-a-centos-6-vagrant-base-box-from-scratch-using-virtualbox.html

https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux_OpenStack_Platform/3/html/Installation_and_Configuration_Guide/Disabling_Network_Manager.html
