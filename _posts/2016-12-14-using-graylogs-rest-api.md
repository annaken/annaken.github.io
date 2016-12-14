---
layout: post
title:  "Using Graylog's Rest API"
tags:   [technical, graylog]
date:   2016-12-14
comments: true
---

Lately I've been working on setting up a logging stack consisting of the ELG components - that is, ElasticSearch, Logstash, and Graylog. Obviously we're doing automated installs of everything, but in the case of Graylog it turned out that the easiest way to configure most of the components was to do calls to Graylog's own API.

In the Graylog UI, the API can be found at System / Nodes and then clicking on the API browser button for any of your nodes. This takes you to the Swagger API browser, where you can click on various API endpoints to see what the options are and how to call them. The API needs authentication because here you can do all of the operations, dangerous or not.

Not everything is fully documented, so a bit of creativity is helpful - try doing some manualy changes, seeing what you did with a GET and then feeding it back in with a POST.

Here are a handful of API calls that we needed to do to set Graylog up with a generic message stream, and with some basic users, roles, and LDAP authetication.

## Set up a stream to handle all incoming messages

To capture all messages we'll filter on *timestamp exists*, assuming that all valid messages will have a timestamp.

It's not immediately obvious from the API browser what the contents of *rules* should be, but with a bit of reverse engineering it turns out that "type: 5" means the field exists.

The "value: 1" field is required by the API call, but really it's a relic of the other sorts of filters - if we were matching on a string or a regex, the *value* field is where it would go.
Here we're not matching on anything so we just need a placeholder value.

    POST /streams
    {
      "title": "All messages",
      "description": "All messages are routed here",
      "matching_type": "OR"
      "rules": [
        {
          "field": "timestamp",
          "type": 5,
          "value": "1",
          "inverted": false
        }
      ],
      "content_pack": null,
    }

## Create a simple user

We're going to set up LDAP shortly, but there might be cases where you want to install a single user either as a test case while you're just getting going, or for a machine user. Here we install a user who only has permissions to read metrics so that we can hook this up to Prometheus for monitoring.

    POST /users
    {
      "username": "metrics",
      "password": "metricsuserpassword",
      "email": "metrics@example.com",
      "full_name": "Queen Metrics",
      "permissions": [
        "metrics:read"
      ],
      "timezone": "UTC"
    }


## Create a user role

Graylog comes with just a couple of roles pre-configured, Admin and Reader. To create something in-between, we'll create a Developer role with enough permissions to do fun stuff but not destroy the world.

    POST /roles
    {
      "name": "Developer",
      "description": "Developer role",
      "permissions": [
        "streams:read",
        "streams:edit:*",
        "streams:create",
        "dashboards:read",
        "dashboards:edit:*",
        "dashboards:create"
      ],
      "read_only": false
    }

## Set up LDAP

For all that this has the most variables to set, this is otherwise straightforwards.
If you're having a lot of trouble it's often easier to experiment in the UI and then do a GET to the rest API to retrieve a working config.

    PUT /system/ldap/settings
    {
      "enabled": true,
      "system_username": "cn=ldapadapter,ou=users,dc=mydc",
      "system_password": "password",
      "ldap_uri": "ldap://example.com:10389/",
      "use_start_tls": false,
      "trust_all_certificates": false,
      "active_directory": false,
      "search_base": "ou=users,dc=mydc",
      "search_pattern": "(&(objectClass=inetOrgPerson)(uid={0}))",
      "display_name_attribute": "cn",
      "default_group": "Reader",
      "group_mapping": {},
      "group_search_base": "",
      "group_id_attribute": "",
      "additional_default_groups": [
        "Developer"
      ],
      "group_search_pattern": ""
    }

## Profit

Go automate!
