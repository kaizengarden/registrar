---
title: registrar: registration agent for binder
markdown2extras: tables, code-friendly
---
<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2014, Joyent, Inc.
-->

# registrar

This is the reference documentation for registrar, which provides a daemon
that maintains entries in ZooKeeper for the current host.  You should be
familiar with how `binder` works to understand this, as that documents the way
the system works.  This codebase is miniscule, and only serves to get the
host on which the daemon runs registered in ZooKeeper.

# Overview

Recall that binder expects host/service entries to be in a JSON payload under
a reversed path from the DNS name.  That is `foo.joyent.com` is expected to be
under `/com/joyent/foo` in ZooKeeper.  The registar agent is a stand-alone
piece of code that maintains "service" entries and leaf entries in ZooKeeper
as appropriate.  All you are really expected to do with this repo is bundle
it into your Manta appliance and configure it appropriate at deployment time.

So, for example, you're deploying a "storage node" into Manta; that is, a
standalone box for which we only want to return A records from DNS.  All you
need to do is have the config file for registrar, which is located at
`/opt/smartdc/registrar/etc/config.json`, look like:

    {
        "registration": {
            "domain": "6b153fc6-1d3c-434a-ba43-daaa8d474fdf.stor.sds.us-east-1.joyent.com",
            "type": "host",
            "host": {
                "address": "10.88.88.130"
            }
        },
        "zookeeper": {
            "servers": [ {
                "host": "10.99.99.204",
                "port": 2181
            } ],
            "timeout": 1000
        }
    }

And start the agent.  You should generate that config file at deployment time.

# Health Checking

The registrar agent supports the ability for you to run an arbitrary command on
an arbitrary interval, and check the exit status and/or the stdout contents with
a (JS) regular expression.  After a defined number of failures, registrar will
remove the nodes that it created in ZooKeeper, thereby removing the zone from
DNS; when health checks begin to pass again it will recreate them.  Note that
there is no "cross-fleet" integration, so there is a risk here that all zones
in a given service might take themselves out of DNS if an upstream is in a bad
place.  That said, most times one or two boxes are down, and removing those
systems is all upside for the service.  Note that health checking is optional,
but encouraged.

# Configuration

There are only two sections to the registration config file `registration` and
`zookeeper`. `zookeeper` should match exactly what
[zkplus](http://mcavage.github.com/node-zkplus) specifies.  `registration` at
minimum needs a `domain` field, which is the DNS name to register this host
under and the type+config block for that leaf node, like the example above.

Lastly, the health check in this example is silly, and just runs the date
command, checking to see if the date is at the `03` or `05` minute of the hour,
in which case, things are ok, otherwise health checks will fail.  But the
parameters are what matters.

Additionally, you can/should specify a service section when the current type
is a `load_balancer` to define the *non-leaf* service entry.  You can
technically do this from anywhere, but by convention and for sanity we only
do it from load balancers.  In which case, the config might look like:

    {
        "registration": {
            "domain": "1.moray.us-east-1.joyent.com",
            "type": "load_balancer",
            "load_balancer": {
                "address": "10.99.99.193"
            },
            "service": {
                "type": "service",
                "service": {
                    "srvce": "_http",
                    "proto": "_tcp",
                    "ttl": 60,
                    "port": 80
                }
            }
        }
        "zookeeper": {
            "servers": [ {
                "host": "10.99.99.40",
                "port": 2181
            }, {
                "host": "10.99.99.192",
                "port": 2181
            } ],
            "timeout": 1000
        },
        "healthCheck": {
            "command": "/opt/local/bin/date -u",
            "ignoreExitStatus": false,
            "stdoutMatch": {
                "pattern": "^\\w+\\s+\\d*,\\s+\\d{4}\\s+\\d{2}:0[35]:\\d{2}\\s+[AP]M\\s+UTC"
            },
            "interval": 1000,
            "threshold": 3,
            "timeout": 1000
        }
    }

While the service subsection is annoyingly verbose, that's exactly what it's
expected to look like in binder, so meh. That's what it is here.
