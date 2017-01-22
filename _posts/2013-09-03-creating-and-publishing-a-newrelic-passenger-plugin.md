---
layout: post
title: "Creating And Publishing A New Relic Passenger Plugin"
excerpt: "It is suprisingly easy to get passenger stats from the built in commands to show up in newrelic dashboards."
description: ""
category: 
tags: [monitoring, ruby, passenger]
---
{% include JB/setup %}

[New Relic](http://newrelic.com/) Plattform now allows custom agents to
report performance metrics. There exist several SDK to pick from as well
as an web API.

We use [Phusion Passenger](https://www.phusionpassenger.com/‎) web server
to deploy our Rails apps. New Relic already helps a lot in fixing
performance issues inside the apps. Since we started using the New Relic
server monitoring agents our previous setup has become somewhat
obsolete. Prior we were running munin to gather statistics on our
servers and applications. We were using the default ubuntu 12.04
packages with a custom theme (the built in one is just terrible). To
monitor our rails apps we used the
[munin-plugin-rails](https://github.com/barttenbrinke/munin-plugins-rails)
gem which gathers data on

-   rails apps by parsing the log files
-   passenger web server by using the provided command line utilities

Parsing rails logs does provide some basic information on the
applications performance. But it makes changing the Rails log formant
difficult (and Rails log format is NOT production ready) and of course
New Relic is just so much nicer to look at :)\
The information gathered for passenger was still quite useful and not
available in New Relic but munin is just really terrible if you actually
want to do more than just have a quick look at the long term development
(drilldown, setting timeperiods etc. is not possible). Also we wanted to
know how much memory was used per Rails-App (we have a multiapp deploy
per passenger webserver). So either we would have to add code to the
munin plugins ... or try out the (kinda) new New Relic plugin
architecture.

And yes - it is suprisingly easy to write a custom New Relic agent.

1.  Download Ruby example agent
2.  add some code to gather data (here easy: copy from munin gem)\
    `passenger_status_output =~ /count\s+=\s+(\d+)/`
3.  send data to New Relic\
    `report_metric "passenger.processes.running", "Processes", $1`
4.  start agent on your server (add your license key first)
5.  Metrics start to show up in New Relic UI
6.  Adjust UI (click, drag, drop, finished)

![Screenshot showing editing the plugin dashboard]({{ site.url }}/assets/images/newrelic-editing-plugin-dashboard_640.png "Screenshot showing editing the plugin dashboard")

Later you can publish the Agent ... I have the feeling New Relic makes
this harder than it should be. Most likely because there is no QA for
the plugins listed in their plugins directory. I feel it would be easier
if the UI configuration could be stored as a config file and versioned
alongside the agent code. This would allow useful sharing of agents
without the need of publishing them (i.e. if they really are just a
hack - as this one :).

Anyway, reading the [publishing
documentation](https://newrelic.com/docs/plugin-dev/publishing-your-plugin-to-plugin-central)
makes you think it is real hard to publish a plugin. You need a company
site, contact and support information and a whole load of requirements.

But actually you need no more than to host your plugin on github (or
similar). This provides you with a webpage (github readme) a contact and
support possibility (github issue page) and download link (github
repository as zip link). Also the actual publishing process is a simple
one page form ...

![Screenshot of publishing form for a on New Relic plugin]({{ site.url }}/assets/images/newrelic-publishing-plugin_600.png "Screenshot of publishing form for a on New Relic plugin")

And after accepting the [terms of service
stuff](https://rpm.newrelic.com/terms_of_service/developer) (#TODO: Read
them) the plugin appears at the bottom of the plugins directory

![Screenshot of plugins directory with the newly created plugin at the bottom]({{ site.url }}/assets/images/newrelic-plugins-repository-with-freshly-published-plugin_640.png "Screenshot of plugins directory with the newly created plugin at the bottom")

