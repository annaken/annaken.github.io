---
layout: post
title:  "Setting up Request Tracker 4"
tags:   [technical, request tracker, exim]
date:   2013-08-14
---

Request-Tracker 4 is a powerful ticketing system but setting it up and making it work was a bit of an epic task. 

The official wiki isn't bad, and the mailing list has a lot of answers to a lot of questions, but I still had to do a lot of experimentation to make things work happily together. So here's a how-to about what I did. It's massive, I'm sorry.


## Installing the basics

Install a fresh copy of debian squeeze, plus any extras eg vim sudo curl

Add the squeeze backports to be able to get the latest version of rt

    vim /etc/apt/sources.list
      deb http://backports.debian.org/debian-backports squeeze-backports main
    $ apt-get update

Install a database: I chose mysql (vs postgresql and sqlite). Remember the password you assign!

    $ apt-get install mysql-server
    $ apt-get install rt4-db-mysql

Make sure it is up and running

    mysql -p

Install a webserver: I chose apache2 (vs nginx)

    aptitude install rt4-apache2

Make sure it works

    curl localhost:80

Install request-tracker itself.

    aptitude install request-tracker4

At this point in my install, aptitude declared that it needed some dependencies that it didn't intend to install by default, which would leave RT uninstalled. At the prompt of "accept this solution" I said no, and was then invited to install a whole bunch of things, which I said yes to.

Part of the configuration during this process was to enter the following:

* name for the RT instance? myRTserver
* handle site config permissions? yes (then it went off to install the whole world)
* configure db with dbconfig-common? yes

At this point you'll need to supply the root password you installed mysql with, and set up new passwords for RT to access mysql.


## Configure Apache

Configure apache to use a cgi handler. I chose modperl (vs. fastcgi).

The modular Apache setup puts configs under /etc/apache2/sites-available, and then symlinks to them from sites-enabled to make them live. So we need to make a config for RT under /etc/apache2/sites-available/tickets.company.com. This is the basic setup, authenticating against htpasswd. I'll cover AD authentication at a later date.

    <VirtualHost *:80> 
      ServerName tickets.company.com
      ServerAdmin systems@company.com
      AddDefaultCharset UTF-8
      PerlSetEnv RT_SITE_CONFIG /etc/request-tracker4/RT_SiteConfig.pm
      Alias / /usr/share/request-tracker4/html
      Errorlog /var/log/apache2/error.log
      Transferlog /var/log/apache2/access.log

      <Perl>
        use Plack::Handler::Apache2;
        Plack::Handler::Apache2->preload("/usr/share/request-tracker4/libexec/rt-server");
      </Perl>

      <Location />
        SetHandler modperl
        PerlResponseHandler Plack::Handler::Apache2
        PerlSetVar psgi_app /usr/share/request-tracker4/libexec/rt-server
        AuthType Basic
        Require valid-user
        AuthName "Restricted Resource"
        AuthUserFile /etc/apache2/htpasswd
      </Location>
      <Location /NoAuth> # for email
        allow from all
        satisfy any
      </Location>
    </VirtualHost> 

I also found it was necessary to set in 

    /etc/apache2/apache2.conf
      MaxRequestsPerChild 1000

as when it is set to 0, a process will never expire and eventually eat all the memory.

Restart apache, and navigate in a browser to `http://tickets.company.com` and the rt login screen should be presented. You ought to be able to log in as root with the mysql root user password that you set.


## Basic RT configuration

Now that we have Apache configured and working,  we can set up the basics of RT mostly via the web interface.

### Make a super-user

To make the first super-user, you need to be logged in as root; you’ll have to have apache configured to authenticate against htpasswd (see example config above).
Make a new group called, for example, ‘Super-Users’.
Add the user you want to be made super-user to this group.
Go to Tools / Configuration / Global / Group rights, select the Super-Users group on the left-hand side, and under ‘rights for administrators’ select ‘do anything and everything’.
Now you should be able to log out as root, and log in as your new super-user. If the new super-user exists in ldap rather than htpasswd, remember to switch out that part of the apache config.

Note: Logging out
Using apache, I found I was unable to logout - I would click on the logout button, and be immediately logged back in again. In actual fact, using this apache configuration, where authentication is either via a list in htpasswd or with ldap, RT should not present the logout button at all. It can be disabled in RT_SiteConfig.pm by setting
Set($WebFallbackToInternalAuth, 0);
To actually logout, clear the browser cache, and the login box asking for your credentials should be displayed.

### Groups and Queues

The first bit of configuration you’ll need to do is make some groups, so that when you add users you have somewhere to put them.
“Tools / Configuration / Groups / Create”
enter the name and description of each new group.
We also need to make some queues so that the tickets can go somewhere.
“Tools / Configuration / Queues / Create”
enter the name and description of each new group.
There are a bunch of other fields, but you probably don’t need to worry about them right now. Just make sure the ‘enabled’ box is ticked so that your queues are active.

For any given queue, you’re likely to want to assign the following rights:
create ticket (by email or ui)
reply to a ticket
view queue (so they can see their ticket when they log into the selfservice UI)
view ticket summary (so they can see the ticket details in the UI)
You can do this by going to each queue, selecting “Everybody” and then assigning rights, or if want to grant the same rights to every queue, assign them via “Tools / Configuration / Global / Group rights”.
A point to note: if you apply rights here they DO NOT show up as ticked at the queue level, even though they are applied.

You’ll also probably want to grant additional rights to the group that will be actioning the tickets. Still under “Tools / Configuration / Global / Group rights”, add the group on the left, and then select appropriate rights. Which will probably be most things. 

The most important ones to grant are probably
own ticket
take ticket
delete ticket (in case of spam)

The requestor(s) of a given ticket will want to be given special rights over that ticket. (A requestor is a person who is waiting for the ticket to be resolved.) Still under “Tools / Configuration / Global / Group rights”, select “Requestor” and grant
reply to ticket

Now test making a ticket using the interface on the homepage dashboard.

### Users

You can create users in a similar manner to groups and queues, ie
“Tools / Configuration / Users / Create”
making sure that the apache configuration uses “htpasswd” to store the users and their hashed passwords.

Test that a given user can access / not access the various pages as appropriate. Top tip: to save having to clear your cache to log in as another user, use a different browser.


## LDAP Authentication

To authenticate users against LDAP, the Apache config needs tweaking as follows:

    <Location />
      SetHandler modperl
      PerlResponseHandler Plack::Handler::Apache2
      PerlSetVar psgi_app /usr/share/request-tracker4/libexec/rt-server
      AuthType Basic
      Require valid-user
    
    # Comment out the htpasswd section
    #    AuthName "Restricted Resource"
    #    AuthUserFile /etc/apache2/htpasswd
    
    # Replace it with LDAP authentication
      AuthName "company.com AD"
      AuthBasicProvider ldap
      AuthzLDAPAuthoritative off
      AuthLDAPGroupAttributeIsDN on
      AuthLDAPBindDN "CN=apache,CN=Users,dc=company,dc=com"
      AuthLDAPBindPassword "your password here................"
      AuthLDAPURL "ldap://dc.company.com:389/CN=Users,DC=company,DC=com?mail?sub?(objectClass=*)"
    </Location>

The advantage to using LDAP over htpasswd is that (a) if you have more than a couple of users to add it's much faster and (b) if LDAP-authenticated users email a ticket to RT their user profile will be automatically set up.


## Exim setup

Ultimately we need to get our email client/MUA to send an email to its MTA which sends it to the RT server which gets picked up by exim which sends it to rt-mailgate which sends it to rt to make a ticket.
That should be easy, right?

### Configure exim to send and receive email

First of all, the MTA needs to be set up. Since I'm using Debian Squeeze I will be configuring exim4, which comes installed.

To get your bearings, type `exim -bV` or `update-exim4.conf -v` to see where exim is reading its config from. Since I've not touched any settings yet, this directs me to /var/lib/exim4/config.autogenerated, which is generated from the config files in /etc/exim4/.

To configure exim, start by using dpkg-reconfigure:

    dpkg-reconfigure exim4-config

I entered the following:

* type of mail config: mail sent by smarthost, received by SMTP
* system mail name: same as the server name, tickets.company.com
* IP addresses to listen on: 127.0.0.1; ::1
* other domains for which mail is accepted: tickets.company.com
* machines to relay mail for: blank
* outgoing smarthost: smtp.company.com
* hide local mail name: no
* keep number of dns queries minimal: no
* mailbox in /var/mail
* split configuration: yes

At the end of the configuration, exim will restart.

Test the new configuration:

    exim -bt anna@company.com

which ought to return details of the router and mail server that will be used to deliver mail to this address.

Next, test that exim can deliver an email to this address:

    echo 'hello' | mail -s "Test subject" anna@company.com

Next we'll look at configuring RT and rt-mailgate to interact with exim.


## Configure rt-mailgate

rt-mailgate is the mail gateway which RT uses to convert emails to tickets. Before we get stuck into configuring it, first a few things need configuring in RT.

The RT configuration file can be found at `/etc/request-tracker4/RT_SiteConfig.pm`.
It's best not to modify this file directly, but to add config fragments under RT_SiteConfig.d and then recompile. To ready RT for email, we do the following:

    /etc/request-tracker4/RT_SiteConfig.d/50-debconf :
      # THE BASICS:
      Set($rtname, 'rt.tickets.company.com');
      Set($Organization, 'company.com');
      Set($CorrespondAddress , 'anna@company.com');
      Set($CommentAddress , 'anna@company.com');
      # THE WEBSERVER:
      Set($WebPath , "/rt");
      Set($WebBaseURL , "http://tickets.company.com");

    /etc/request-tracker4/RT_SiteConfig.d/60-mailconf :
      Set($SendmailPath, "/usr/lib/sendmail");    
      Set($SendmailArguments, "-t");
      Set($OwnerEmail, "anna@company.com"); #who to email errors to

Then run

    update-rt-siteconfig-4

to update RT.

Now we can look at configuring rt-mailgate itself. First, let's make sure rt-mailgate works in its own right.

    echo "Test123" | rt-mailgate --queue General --action correspond --url http://tickets.company.com

If this gave you a 403, then you'll need to configure `apache2-modperl2.conf`. There should be a stanza about limiting access to mail gateway. Since it's NoAuth anyway, let's just allow access across the board for it:

    /etc/apache2/sites-available/apache2-modperl2.conf
      <Location /rt/REST/1.0/NoAuth>
        Order Allow,Deny
        Allow from all
        Satisfy any
      </Location>

Add the line "Allow from rt.tickets.company.com" and restart Apache.

To configure exim to forward emails to rt-mailgate, edit /etc/aliases :

    rt: "|/usr/bin/rt-mailgate --queue general --action correspond --url http://tickets.company.com/rt"
    rt-comment: "|/usr/bin/rt-mailgate --queue general --action comment --url http://tickets.company.com/rt"

In my installation, I decided to make queue names which were modular, with a 'from' and a 'to' part, of the shape 'from@to' so that I could use email addresses that look like from@to.company.com. To set this up, I added two files to exim, one a list of requestors, and one a list of recipients:

    /etc/exim/queuenames/list.of.requestors (the bit before the @) :
      account-managers
      developers

    /etc/exim/queuenames/list.of.recipients (the bit after the @) :
      systems.company.com
      dba.company.com

NB make sure that the two lists combine to make the queue names that are configured in the RT gui.

Also to note - if you add to the list of requestors then you don’t need to reconfigure exim, but if you add to the list of recipients you’ll need to do dpkg-reconfigure exim4-config again and add in the extra subdomain as a destination to receive mail for.

Alternative - and probably easier to keep track of - these settings are all saved in /etc/exim/update-exim4.conf.conf. If you do add to the list of requestors you can add in the additional domains here, under dc_other_hostnames, making sure to separate your items with colons (no csv here, it will silently fail, madness lies beyond). 
Then run update-exim4.conf (yes, that's the command), and then restart exim.

Also to configure to make this work: 

    /etc/exim/conf.d/main/01_exim4-config_listmacrosdefs
      RT4_URL = http://tickets.company.com/

    /etc/exim/conf.d/router/950_exim4-config_rt
      request_tracker4:
       driver            = redirect
       domains           = /etc/exim4/queuenames/list.of.recipients
       local_parts       = /etc/exim4/queuenames/list.of.requestors

       local_part_suffix = -comment
       local_part_suffix_optional
       pipe_transport    = request_tracker4_pipe
       data              =   "|/usr/bin/rt-mailgate-4 \
                             --queue \"${local_part}@${extract{1}{.}{$domain}}\" \
                             --action ${substr_1:${if eq{$local_part_suffix}{} \
                             {-correspond}{$local_part_suffix}} } \
                             --url RT4_URL"
       user              = www-data

    /etc/exim/conf.d/transport/30_exim4-config_rt_pipe
      request_tracker4_pipe:
      driver         = pipe
      return_fail_output
      allow_commands = /usr/bin/rt-mailgate-4

Recompile the config

    dpkg-reconfigure -plow exim4-config

Test making a ticket via email from the command line of the exim server.

    echo "This is a test ticket" | from=anna@company.com mail -s "test $(date)" accountmanagers@systems.company.com

If these tests all went smoothly and the server is able to email out, then next we need to configure RT to interact with users by email.

NB here I am using swaks to test RT so that I can send emails internally without needing to open exim up to receive emails from gmail (and therefore spam). A basic swaks command looks like
swaks --to accountmanagers@systems.company.com --from anna@company.com --server tickets.company.com -d "Subject: This is the subject line \n\n Here is some text that should go into the body"
Telnet would also work, but is a bit more long-winded.

### More testing:  

Test making a ticket via email from another internal server (this allows us to test before the smtp port is opened to the whole world:

    swaks --to accountmanagers@systems.company.com --from anna@company.com --server tickets.company.com

A person not in the domain should not be able to make a ticket

    swaks --to accountmanagers@systems.company.com --from abadperson@yahoo.com --server tickets.company.com
    Action: no ticket created
    /var/log/request-tracker/rt4.log:  RT could not load a valid user, and RT's configuration does not allow for the creation of a new user for your email. Could not record email: Could not load a valid user


An email sent to a non-existent queue should not create a ticket

    swaks --to systems@wrongaddress.com --from anna@company.com --server tickets.company.com
    Action: no ticket created
    /var/log/exim4/mainlog: Connection refused
    exim -Mvh <emailID>: spam score of -1

Mail gets returned to sender with the message “A message that you sent could not be delivered to one or more of its recipients. This is a permanent error. The following address(es) failed: systems@systems.whereever.com retry timeout exceeded”


From a valid, privileged user in RT who is not a user in AD:

    swaks --to accountmanagers@systems.company.com --from bobby.tables@company.com --server tickets.company.com
    Action: ticket created, but user cannot access selfservice as no password set.


From a valid AD user who is present but unprivileged user in RT

    Action: ticket created. User can access selfservice.


From a valid AD user who is not in RT:
    
    Action: ticket created, unprivileged user created. User can access selfservice but not see ticket


As root, make this new user privileged.

    Action: User can now access selfservice and see ticket, as well as the queue that ticket was sent to.


## Useful bits and bobs

If you've made it this far, good on you. Personally, I wanted to punch RT in the face by this point, but you'll be pleased to hear you're just about done.

I added spamassassin to exim, but I don't think I did anything particularly creative with it, so I'll leave you to go research that bit elsewhere.

As I finished my installation and started to implement RT for our users, I came across lots of little bits and pieces that were not immediately obvious to me. Maybe they'll be obvious to you, in which case skip this post and go revel in the glory of having a working ticketing systems. If not, here's a brain dump of them.


### Clearing the test database

You filled your database with users called "Ivor Bigun"? No worries, just drop the database and remake it:

    mysql -u root -p
      DROP DATABASE rtdb;
    rt-setup-database-4 --action init --dba root


### List all users

The user list by default only shows priviledged users.
If you do “Find all users whose name matches %” this will list everyone who has “Let this user access RT” enabled.
If you do “Find all users whose name matches %” and tick “Include disabled users”, this will list all the users in the system, even those who have neither access nor rights granted.


### Emailing tickets

When a ticket is created by email, the subject is set as the subject of the email.
When replying to a ticket by email, the subject must include a string along the lines of “[tickets.company.com #40]” but it doesn’t matter what else it contains, or which queue email address it is sent to, it will still be added to ticket #40.


### Replying vs commenting

You can either reply to a ticket, or put a private comment on it. Just to make it extra obvious which of those you’re doing, the message box is RED for a public reply, and WHITE for a private comment.


### Saved Searches

“RT System Saved Searches” are only available to super-users.


### Deleting tickets

Only tickets that have ‘new’ status can be deleted - so if needs be, change the status first, and then delete.


### Setting the default queue

In /usr/share/request-tracker4/html/Elements/SelectQueue , set the value $Default to the queue number (show in the UI).
NB the “New ticket in” quick create button on the homepage sets its own default as the first queue alphabetically that the user has rights to (via their group membership).


### Using RT-crontool

Schedule automatic ticket actions using rt-crontool. For example, I want all of the tickets in the ‘projects@systems’ queue that have had no updates in two weeks (336 hours) to have their status changed to ‘stalled’.

    rt-crontool   --search RT::Search::ActiveTicketsInQueue  --search-arg projects@systems  --condition RT::Condition::UntouchedInHours --condition-arg 336 --action RT::Action::SetStatus --action-arg stalled

    
### Error: Message body not shown because it is too large
 
Edit 

    /etc/request-tracker4/RT_SiteConfig.pm
      Set($MaxInlineBody, 100000);


### Error: apache memory issues

If a ticket is created by email but in the details of the ticket in the RT UI there is the error “Sending the previous mail has failed”, and in rt4.log there is the error “Could not send mail: Cannot allocate memory”, then make sure MaxRequestsPerChild in apache2.conf is not set to zero - this means processes never expire. I set this to a thousand.


### Hacking the backend but don't see your changes implemented yet? 

Clear the cache with
rm -rf /var/cache/request-tracker4/mason_data/obj/*

That's it.
I hope some of that was useful to you. RT is probably too powerful for its own good, meaning it's quite hard to get off the ground, but is worth the effort.
