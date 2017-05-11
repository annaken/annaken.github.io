---
layout: post
title:  "Testing Packer builds with Serverspec"
tags:   [technical, testing, packer, serverspec, aws]
date:   2017-05-10
comments: true
---

Lately I've been working on building base AMIs for our infrastructure using [Packer](https://www.packer.io/), and verifying these images with [Serverspec](http://serverspec.org). In the opening stages my workflow looked like:

1. Build AMI with Packer
2. Launch instance based on AMI
3. Run Serverspec tests against instance

This works fine, and could potentially be converted into a Jenkins pipeline, but it feels a bit clunky. My AMI is based on a running source instance, why can't I test the instance before the final conversion to AMI?

My preferred pipeline would look like:

1. Build AMI with Packer
2. As the final build step, run Serverspec
3. If the tests fail, abort the build

That way all of our produced AMIs would be tested and verified.

## Running Serverspec from within the source instance

Packer offers a way to poke shell commands into the AMI as part of the 'provisioners' stage, and this can be used to kick off the testing like:

    "provisioners": [
      ...
      { "type": "shell",
        "inline": [
          "sudo apt-get install -y ruby",
          "sudo gem install serverspec",
          "cd /tmp/tests",
          "rake spec TARGET_HOST=localhost"
        ]
      }

However, one of the nice things about Serverspec is that it doesn't require anything to be installed on the target host. Running tests like this means we have to install ruby and serverspec on our shiny new server and I wasn't really down with that.

## Running Serverspec remotely

Looking at the Packer docs more closely, it turns out there is a new 'shell-local' command introduced for [exactly this reason](https://github.com/hashicorp/packer/pull/1823), so we can run Serverspec from our local machine before the AMI is finalised.

The provisioners section of the Packer json file would look like:

    "provisioners": [
      {
        # whatever steps you normally do to set up the instance
      },
      { "type": "shell",
        "inline": [
          "curl http://169.254.169.254/latest/meta-data/local-ipv4"
        ]
      },
      { "type": "shell-local",
      "command": "export SSH_USER=ubuntu && cp -r ../tests/* . && egrep -m1 -o '[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}' build.log | xargs -I host rake spec TARGET_HOST=host"
      }
    ]

Let me talk you through those commands.

First, I should explain that when I call Packer, I do

    packer build -machine-readable myami.json | tee build.log

so that the Packer output gets formatted and sent to a build log. This is going to let me call commands and read back the results.

### curl http://169.254.169.254/latest/meta-data/local-ipv4

Amazon provides a way to find out metadata about an instance from within the instance, by [curling a special endpoint](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html).

By doing this call from within the instance, I get the IP printed out to build.log, which I'll use in the next step.

### export SSH_USER=ubuntu

Serverspec needs to ssh into the instance to run the tests. To this end, in the "builders" section of the Packer json file, I specified a keypair (which was already planted on AWS) like:

      "ssh_username": "ubuntu",
      "ssh_keypair_name": "ubuntu-deploy",
      "ssh_private_key_file": "/home/annaken/.ssh/ubuntu-deploy",

### cp -r ~/serverspec/* .

Yeah, so rake only runs if you're standing in the same directory as the Rakefile. There are other ways around this but just copying the contents of my whole servrspec dir was the easiest fix for me.

### egrep -m1 -o '[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}' build.log

Having run the curl to fetch metadata, the IP is outputted the IP. Now I just grep for that IP (remembering to double-escape the dots because we're in JSON).

### xargs -I ip rake spec TARGET_HOST=ip"

I use xargs to pass the instance IP through to rake, which runs my set of serverspec tests against the instance at the given IP.

Note that I've slightly amended my Rakefile to allow target host IP to be passed in as an environment variable.

## Fail the build on test failure

Serverspec runs from my local machine against the source instance, rattling through the tests and reporting back on the failures.

If my tests fail, then I want to abort the build, which is easy to implement because if Packer receives any failing exit codes from its provisioners then it will abort by default (this behaviour is overridable with the debug flag).

So all I need to do is write a test that will give a non-zero exit code if any tests fail.

If all my tests pass I'll get a line in build.log that looks like

    505 examples, 0 failures

so this can be my test - grep for the existance of this "0 failures" string. However, I have to be a little bit sneaky: doing a simple "grep '0 failures'" always passes, because the grep expression itself is outputted to build.log before the grep is executed!

    Executing local command: grep ' 0 failures' build.log

Meaning it always matches, and always passes. To get round this I did:

    {
      "type": "shell-local",
      "command": "egrep ' [^1-9] failures' build.log"
    }

If I get any non-zero number of failures reported, the build will abort.

## Tidy up

I had to plant keys on the source instance so that I could run Serverspec against it; I remove these as a final step:

    {
      "type": "shell-local",
      "command": "rm -rf /home/ubuntu/"
    }

That's it - if all the tests pass, then the AMI is built and available for use very shortly.


### References:

* https://stelligent.com/2016/08/17/introduction-to-serverspec-what-is-serverspec-and-how-do-we-use-it-at-stelligent-part1/
* https://www.morethanseven.net/2014/01/01/testing-packer-created-images-with-serverspec/
* http://code.hootsuite.com/build-test-and-automate-server-image-creation/
* https://github.com/hashicorp/packer/pull/1823
* https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html
