# Building a bastion

Bastion, jump box. Public facing therefore needs to be secure.

https://www.cyberciti.biz/faq/linux-bastion-host/
" Install minimum operating system. Avoid installing desktop software or other apps such as MySQL, Apache and other software."

https://docstore.mik.ua/orelly/networking/firewall/ch05_08.htm :

" Secure the machine.
Disable all nonrequired services.
Install or modify the services you want to provide.
Reconfigure the machine from a configuration suitable for development into its final running state.
Run a security audit to establish a baseline.
Connect the machine to the network it will be used on."

I want to make an AMI specifically for a bastion.

# Disable all nonrequired services.

That sounds easy.

## Remove all non-essential packages from ubuntu

Start with ubuntu minimal server.
All packages on the system are classed as important, required, standard, optional or extra.
If I just remove optional and extra packages that should leave me with just the essentials. Right?

https://askubuntu.com/questions/79665/keep-only-essential-packages

dpkg-query -Wf '${Package;-40}${Priority}\n' | awk '$2 ~ /optional|extra/ { print $1 }' > all_optional_extra_packages

cat all_optional_extra_packages | xargs -I % sudo apt-get -y purge %

Result: catastrophe. System breaks in a million different ways, different every time, often doesn't reboot

## Remove all non-essential packages, apart from ones we think we should keep

Review the list of optional and extra packages.
Oh, grub is optional? That would probably explain the not rebooting
The cloud-init facilities probably need to stay, as we're building a cloud image in AWS
Let's not remove any libraries, for now
Openssh server is optional... yeah we need that
Don't remove resolveconf. Ubuntu doesn't like it
The ubuntu-server package is a meta-package. It's marked as optional but it depends on all the other packages in the ubuntu server system.
And on... and on...

Result: still unstable, occasionally unbootable.
Ok, let's stop trying to uninstall hundreds of interconnected packages we don't know inside out

## Make a list of packages we really really don't want on a bastion

Our list in theory:

bash-completion
curl
ed
ftp
gawk
git
nano
netcat-openbsd
net-tools
perl
python2
python3
rsync
screen
tar
telnet
tmux
vim
wget

In practise we can't remove:

curl - needed for consul
net-tools - needed for sshuttle
tar - needed for ansible
perl - needed for ssh
python2 - needed for ansible
python3 - needed for AWS instance healthcheck

## Restrict user functionality

### Change all user shells to /bin/nologin

Can't - breaks sshuttle

### Restrict allowed commands in authorized_keys

Can't - we use sshuttle, not ssh

### Use rbash instead of bash

Can't - breaks sshuttle

## Restrict user privs

No sudo for anyone

## Increase security

Install fail2ban
Add 2FA



# Which OS?

Network Security, Firewalls and VPNs
https://books.google.no/books?id=qZgtAAAAQBAJ&pg=PA264&lpg=PA264&dq=bastion+host+OS&source=bl&ots=FblGJ7dHgY&sig=1n97x5lzpQbs1Muj6I9AhOmtBIA&hl=no&sa=X&ved=0ahUKEwiM19bg9d_TAhUDAZoKHUBlCBkQ6AEIeDAJ#v=onepage&q=bastion%20host%20OS&f=false

Proprietary bastion host OS eg Cisco IOS
or
General purpose OS, locked down







