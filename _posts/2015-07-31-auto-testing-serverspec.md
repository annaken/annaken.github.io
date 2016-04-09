---
layout: post
title:  "Automated server testing with Serverspec, output for Logstash, results in Kibana"
tage:   [technical]
date:   2015-07-31 
---

Whether you're spawning VMs to cope with spikes in traffic, or you want to verify your app works on a range of operating systems, it's incredibly useful to have some automated testing to go with your automated VM creation and configuration.

This is a quick run-down of one way to implement such automated testing with Serverspec and get results back that are ultimately visualisable in Kibana. NB the orchestration of the following steps is beyond the scope of this article - maybe some CI tool like Jenkins, orchestration tool like vRO or some custom software.

Overview:
* Automagically create VMs (AWS, OpenStack, etc)
* Configure the VMs with some config management tool (Puppet, Chef, etc)
* Perform functional testing of VMs with Serverspec 
* Output logs that are collected by Logstash
* Visualise output in Kibana

The first two points are essentially prerequisites to this article: create some VMs and install them with whatever cloud and config magic you like. For the purposes of this article, it doesn't really matter. I'm just going to assume that your VMs are 'normal', ie running and contactable.


## Serverspec basics

Serverspec, if you've not used it, is an rspec-based tool to perform functional testing. It's ruby-based, has quite an easy set-up, and doesn't require anything to be installed on the target servers, just that it is able to ssh into the target machines with an ssh key.

Install and set up a la the Serverspec documentation

    $ gem install serverspec
    $ mkdir /opt/serverspec
    $ cd /opt/serverspec
    $ serverspec-init

This will have created you a basic directory structure with some files to get you started.

Right now we have:

    $ ls /opt/serverspec
    
    Rakefile
      spec/
      spec_helper.rb
      www.example.com/

The default setup of Serverspec is that you define a set of tests for each and every server and then run the contents of each directory against the matching host. However this doesn't really fit the workflow we're setting up here.


## Re-organise Serverspec from host-based to app-based layout

To get started, let's delete the www.example.com directory - we don't want to define a set of tests per host like this, we want to make an app-based layout.

In my opinion, one of the easiest ways to organise the layout for your functional tests is to store it alongside your config management code. With this in mind, let's write a simple ntp test.

Writing a Serverspec test


Our ntp Puppet config is found at, and looks like:

    $ cat /opt/puppetcode/modules/ntp/manifests/init.pp
    
    class ntp {
      package { 'ntp':
      ensure => installed
      }
    }

So alongside this directory we can make a sister Serverspec directory, and put our first test in there:

    $ cat /opt/puppetcode/modules/ntp/serverspec/init_spec.rb
    
    require 'spec_helper'
    describe package('ntp') do
      it { should be_installed }
    end


## Making Serverspec run our test

Now we need to edit the Rakefile to reflect this restructuring:

    $ cat /opt/serverspec/Rakefile
    
    require 'rake'
    require 'rspec/core/rake_task'
    
    $host       = 'www.example.com'
    $modulelist = %w( ntp )
    
    task :spec => "spec:#{$host}"
    
    namespace :spec do
      desc "Running serverspec on host #{$host}"
      RSpec::Core::RakeTask.new($host) do |t|
        ENV['TARGET_HOST'] = $host
        t.pattern = '/opt/puppetcode/modules/{' + $modulelist.join(",") + '}/serverspec/*_spec.rb'
        t.fail_on_error = false
      end
    end 

Yes, we did just hard-code the host name and modulelist to test. Don't worry, we'll switch these out in a bit.
Note that we provide a pattern path with a regex to the directory containing our tests. Essentially when we run this file, we will pick up every test that matches the pattern and run these tests against the desired host.


## Run the test

Now, making sure we are standing in the /opt/serverspec/ directory, we can run

    $ rake spec
    
    Package 'ntp'
      should be installed

Green means that the test ran, and the output was successful. So as it stands, we can test our one www.example.com host with our one ntp test. Great! 


## Rewrite the Rakefile to take command-line options rather than hard-coding variables

Right now, our host identifier and our list of modules to test are hard-coded in the Rakefile. Let's rewrite so these are passed in on the command line.

    $ cat /opt/serverspec/Rakefile

    require 'rake'
    require 'rspec/core/rake_task'

    $host       = ENV['host']
    $modulelist = ENV['modulelist']

    task :spec => "spec:#{$host}"

    namespace :spec do
      desc "Running serverspec on host #{$host}"
      RSpec::Core::RakeTask.new($host) do |t|
        ENV['TARGET_HOST'] = $host
        t.pattern = '/opt/puppetcode/modules/{' + $modulelist.join(",") + '}/serverspec/*_spec.rb'
        t.fail_on_error = false
      end
    end 

Now to run the tests we need to do 
    $ rake spec host=www.example.com modulelist=/opt/serverspec/modulelist

where
    $ cat /opt/serverspec/modulelist
      ntp

The modulelist file can be one you write yourself, or generated from something like a server's /var/lib/puppet/classes.txt. It's a way to narrow down what tests are run against each server, as all modules are not necessarily implemented everywhere.


## Manipulating Serverspec output

Next, we want to look carefully at the output generated by Serverspec so that we can track and visualise our tests. We need to track our data carefully so that we can cope with the results of many different VMs.

Serverspec has a number of output options. The 'documentation' style is what we've seen printed to screen so far; there are also json and html reports. It is possible to get all of these formatting options at once by adding the following line to your Rakefile:

    t.rspec_opts = "--format documentation --format html --out /opt/serverspec/reports/#{$host}.html --format json --out /opt/serverspec/reports/#{$host}.json"

So now we have two files at /opt/serverspec/report: www.example.com.html and www.example.com.json.
The json file is the one we're going to pick up and turn into our log.


## Logging format

If we inspect the contents of the www.example.com.json report, we can see that it is of the format:

    {
        "examples": [
            {
                "description": "should be installed",
                "file_path": "/opt/puppetcode/modules/ntp/serverspec/init_spec.rb",
                "full_description": "Package \"ntp\" should be installed",
                "line_number": 4,
                "run_time": 2.525189129,
                "status": "passed"
            },
        ],
        "summary": {
            "duration": 2.609159102,
            "example_count": 1,
            "failure_count": 0,
            "pending_count": 0
        },
        "summary_line": "4 examples, 0 failures"
    }

Each test is an element in the 'examples' array, and at the end we have a summary and a summary_line.

We're going to pick up every test as a separate json object, insert some identifying metadata, and output each test as a line in /var/log/serverspec.log

Apart from the host and module identifiers, it might also be helpful if we knew, for example, that the OS version of the host was, which git branch it came from, and maybe a UUID unique to a test (which could encompass multiple VMs).

With this in mind, we re-write our /opt/serverspec/Rakefile as follows:

    require 'rake'
    require 'rspec/core/rake_task'
    require 'json'

    # Command line variables
    $uuid   = ENV['uuid']
    $host   = ENV['host']
    $modulelist = File.readlines(ENV['filename']).map(&:chomp)
    $branch = ENV['branch']
    $osrel  = ENV['osrel']

    task :spec => ["spec:#{$host}", "output"]

    # Run the Serverspec tests
    namespace :spec do
      desc "Running serverspec on host #{$host}"
      RSpec::Core::RakeTask.new($host) do |t|
        ENV['TARGET_HOST'] = $host
        t.pattern = '/opt/puppetcode/modules/{' + $modulelist.join(",") + '}/serverspec/*_spec.rb'
        t.fail_on_error = false
        t.rspec_opts = "--format documentation --format html --out /opt/serverspec/reports/#{$host}.html --format json --out /opt/serverspec/reports/#{$host}.json"
      end
    end

    # Edit the serverspec json file to add in useful fields
    task :output do
      File.open("/var/log/serverspec.log","a") do |f|
        # Read in the json file that serverspec wrote
        ss_json = JSON[File.read("/opt/serverspec2/reports/#{$host}.json")]
        puts "/opt/serverspec2/reports/#{$host}.json"
        ss_json.each do |key, val|
          if key=='examples'
            val.each { |test|
              modulename = test["file_path"].gsub(/\/opt\/puppetcode\/modules\//,"").gsub(/\/serverspec\/.*/,"")
              test["module"] = modulename
              insert_metadata(test)
              f.puts(JSON.generate(test))
            }
          end
        end
      end
    end

    # Add in the rest of our useful data
    def insert_metadata ( json_hash )
      json_hash["time"]   = Time.now.strftime("%Y-%m-%d-%H:%M")
      json_hash["uuid"]   = $uuid
      json_hash["hostip"] = $host
      json_hash["branch"] = $branch
      json_hash["osrel"]  = $osrel
    end

Now we can run

    $rake spec host=www.example.com filename=/opt/serverspec/modulelist branch=dev osrel=7.1 uuid=12345

And see in /var/log/serverspec.log

    {"description":"should be installed","full_description":"Package \"ntp\" should be installed","status":"passed","file_path":"/opt/puppetcode/modules/ntp/serverspec/init_spec.rb","line_number":4,"run_time":0.029166597,"module":"ntp","time":"2015-07-31-12:21","uuid":"12345","host":"www.example.com","branch":"dev","osrel":"7.1"}

This log can now be collected by Logstash, indexed by Elasticsearch, and visualised with Kibana.

![Kibana_serverspec](/images/kibana_serverspec.png)
