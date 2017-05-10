# Making a hardened bastion AMI

Start with vanilla Ubuntu (need to support 14.04 and 16.04)

We want to make a super minimal AMI that we can use as a bastion

What packages are installed in the minimal server install?

    dpkg --get-selections

Some of these packages are marked optional or extra

    dpkg-query -Wf '${Package;-40}${Priority}\n' | awk '$2 ~ /optional|extra/ { print $1 }'

Could we remove all optional/extra packages from a system?

    dpkg-query -Wf '${Package;-40}${Priority}\n' | awk '$2 ~ /optional|extra/ { print $1 }' | xargs -I % sudo apt-get -y purge %

Answer: not if we want to have a usable system at the end of it.

Ok, what about a subset of these packages? Some of these 'optional/extra' things do look useful... grub, for example. Let's not uninstall grub. Also resolveconf. Also ubuntu-server.

    cat packages_to_remove | xargs -I % sudo apt-get -y purge %

Nope. Hosed the system again. Interestingly, it breaks in a different way every time I try. Sometimes it dies mid-way through, sometimes after a reboot.

Ok, let's stop uninstalling the world and just pick a list of tools to remove that would be helpful to an attacker. Things like curl, ftp, net-tools, rsync, wget, python, perl.

Uninstall these and proceed with setup.

Setup uses Ansible so we can't uninstall tar or python2.7.

We want to be able to use ssh, so we can't uninstall perl.

We also want to use sshuttle, so we can't unintall net-tools.

Ok so now we have a slightly stripped server. To use this as a bastion host, we need to set up restricted user accounts.

First: install users. This happens from a script (generate-authorized-keys) which

