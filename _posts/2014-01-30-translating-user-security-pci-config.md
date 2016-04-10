---
layout: post
title:  "Translating user security for PCI compliance into configs"
tage:   [technical, config management, security, automation]
date:   2014-01-30
---

As part of our becoming-PCI-compliant project, there are a list of requirements about user security that needed translating from legal-speak into practical actions and ways to implement these actions for our Linux servers. We're using Debian, your mileage may vary.

> "The last 4 passwords must be different"

This is controlled by PAM using the pam_unix.so (or pam_unix2.so) module, which ships as default in Debian. 

In the config file `/etc/pam.d/common-password` the line containing "pam_unix.so" needs to include "remember-4", eg:

    password requisite pam_unix.so obscure use_authtok try_first_pass sha512 remember=4

> "Passwords must contain more than 7 characters, and must be a mixture of upper and lowercase letters, and numbers"

Passwords can be tested with the PAM module `pam_cracklib.so`. On debian/ubuntu, this can be installed with

    $ apt-get install libpam-cracklib

and this will generate an entry in the config file found at /etc/pam.d/common-password along the lines of

    password requisite pam_cracklib.so retry=3 minlen=11 lcredit=1 ucredit=1 dcredit=1 ocredit=1
What we're interested in here is primarily minlen - but it's not exactly the minumum length of the password as you might expect. Rather, it's a total of the number of characters in the password, plus scores from lcredit, ucredit, dcredit and ocredit, where the parameters mean

* lcredit = maximum credit allowed from required lower-case characters
* ucredit = number of required upper-case characters
* dcredit = number of required digits
* ocredit = number of required other characters (non-alphanumeric)

So, if we have the requirement of 8 or more characters including numbers, and upper and lowercase letters, then we can set lcredit, ucredit, and dcredit all to 1, and require minlen = (8+1+1+1) = 11 to ensure this policy is enforced.

> "Passwords must be changed every 45 days"

Aha, an easy one. In `/etc/login.defs` , set

    PASS_MAX_DAYS 45

> "Accounts are made inactive 90 days after last login"

Also straightforwards. In `/etc/default/useradd` , set

    INACTIVE=90

> "Sessions timeout after 15 mins inactivity"

Within `/etc/bash.bashrc`, set

    export TMOUT=900

> "Account locks out for 30 mins after 6 failed login attempts"

For this one, we need to install fail2ban (apt-get install fail2ban), and then create a file at `/etc/fail2ban/jail.local` which contains the following:

    [DEFAULT]
    bantime = 1800
    maxretry =6
    Remember to restart fail2ban after you've made the config change.


References:

http://www.deer-run.com/~hal/sysadmin/pam_cracklib.html

http://www.cyberciti.biz/tips/how-to-linux-prevent-the-reuse-of-old-passwords.html

http://www.cyberciti.biz/tips/linux-check-passwords-against-a-dictionary-attack.html

https://www.digitalocean.com/community/articles/how-to-protect-ssh-with-fail2ban-on-ubuntu-12-04
