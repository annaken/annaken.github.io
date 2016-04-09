---
layout: post
title:  "Recovering from puppet cert clean --all"
tage:   [technical, puppet]
date:   2015-05-08
---

If you just did 'puppet cert clean --all' because reasons and now everything is broken like:

    test-server:~# puppet agent -vt
    Warning: Unable to fetch my node definition, but the agent run will continue:
    Warning: Error 400 on SERVER: Could not retrieve facts for test-server.vm: Failed to find facts from PuppetDB at puppetmaster.example.com:8081: SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed: [certificate revoked for /CN=puppetmaster.example.com]
    Info: Retrieving plugin
    Info: Loading facts
    Error: Could not retrieve catalog from remote server: Error 400 on SERVER: Failed to submit 'replace facts' command for test-server.vm to PuppetDB at puppetmaster.example.com:8081: SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed: [certificate revoked for /CN=puppetmaster.example.com]
    Warning: Not using cache on failed catalog
    Error: Could not retrieve catalog; skipping run

STOP PANICKING: we can fix this.

If you have a backup of the puppetmaster's /var/lib/puppet directory, do a restore and hopefully all will be well.

If not, let's fix the puppetmaster (NB here I'm using a monolithic installation - if your puppetmaster and puppetdbs are on separate machines you'll have to adapt this a little bit).

Cleaning all the certificates means that the puppetmaster's own certificate is missing too, so re-generate it with

    $ puppetmaster:~# puppet cert generate puppetmaster.example.com

Now the puppetmaster has new ssl bits and bobs but puppetdb has the old ones. Clean out the puppetdb ssl directory:

    $ puppetmaster:~# rm -rf /etc/puppetdb/ssl/*

And use the handy ssl-setup script to copy the new ones to the right places

    $ puppetmaster:~# puppetdb ssl-setup
    PEM files in /etc/puppetdb/ssl are missing, we will move them into place for you
    Copying files: /var/lib/puppet/ssl/certs/ca.pem, /var/lib/puppet/ssl/private_keys/puppetmaster.example.com.pem and /var/lib/puppet/ssl/certs/puppetmaster.example.com.pem to /etc/puppetdb/ssl
    Setting ssl-host in /etc/puppetdb/conf.d/jetty.ini already correct.
    Setting ssl-port in /etc/puppetdb/conf.d/jetty.ini already correct.
    Setting ssl-key in /etc/puppetdb/conf.d/jetty.ini already correct.
    Setting ssl-cert in /etc/puppetdb/conf.d/jetty.ini already correct.
    Setting ssl-ca-cert in /etc/puppetdb/conf.d/jetty.ini already correct.

Restart all the things:

    $ puppetmaster:~# service puppetmaster restart
    $ puppetmaster:~# service puppetdb restart

Now, let's fix the nodes.

Start with a test node (preferably not in production), to verify all the steps so far worked as expected.
Remove the existing ssl certs with

    $ test-server:~# rm -rf /var/lib/puppet/ssl/*

Now run puppet manually with

    $ test-server:~# puppet agent -vt
    Info: Creating a new SSL key for test-server.vm
    Info: Caching certificate for ca
    Info: csr_attributes file loading from /etc/puppet/csr_attributes.yaml
    Info: Creating a new SSL certificate request for test-server.vm
    Info: Certificate Request fingerprint (SHA256): 92:A9:A6:B1:88:7B:DB:A7:65:00...
    Info: Caching certificate for ca
    Exiting; no certificate found and waitforcert is disabled

Sign the certificate on the master as usual

    $ puppetmaster:~# puppet cert sign test-server.vm

Now your node should run as usual

    $ test-server:~# puppet agent -vt
    Info: Retrieving plugin
    Info: Loading facts
    Info: Caching catalog for varnish4.vm
    Info: Applying configuration version '1431084513'

The final step is to re-generate certificates for all the rest of your nodes.
Option 1: log into every server and repeat the above.
Option 2: automate option 2 - think ssh, clusterssh, etc

Good luck!

PS I lied - the final, final step is to set up proper backup and restore of your certificate store at /var/lib/puppet/ssl and delete the clean --all line from your command history so you can't accidentally run it again.

References: https://docs.puppetlabs.com/puppetdb/latest/install_from_source.html
