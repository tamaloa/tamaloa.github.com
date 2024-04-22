---
layout: post
title: "SQLite in production"
description: "Using activerecord-enhancedsqlite3-adapter gem in a project to make sqlite a database capable of production"
category: 
tags: [rails, sqlite]
---
{% include JB/setup %}


Following up on Stephen Margheim's excellent talk at 2024 [wroclove.rb](https://wrocloverb.com/) i set out to optimize our sqlite setup.

We are running a really simple app to gather some operational statistics for [samarbeid](https://www.samarbeid.org/about-samarbeid/) instances we operate. To keep it simple we left the default sqlite setup in place and deployed the app as a single docker container. We had the occasional `SQLite3::BusyException: database is locked (ActiveRecord::StatementInvalid)` exceptions but as the statistics reporting was retried on errors we postponed fixing it. But up to recently we had "add proper postgres database" on our ToDo list.


## Step 1 - How to measure

I used [ab (apache-bench)](https://httpd.apache.org/docs/current/programs/ab.html) for decades to do load testing on apps. 
Mainly because it has been available forever in default system packages (and we used to use apache+passenger a lot).
Stephen used [oha](https://github.com/hatoo/oha) in his presentation and i must say it does look a lot nicer. So without 
any deeper comparison let's try it.

    # Checkout locally and build docker container as we do not want to setup a local rust environment
    git clone https://github.com/hatoo/oha.git
    cd oha && docker build . -t oha:latest

We have our local rails app running on port 3000 so now it's a simple `docker run` (use host network to allow localhost adress).
It does have a nicely coloured terminal output and statistics histogram.


    $ docker run -it --net=host oha:latest http://localhost:3000/up
    
    Summary:
      Success rate:	100.00%
      Total:	0.5759 secs
      Slowest:	0.5459 secs
      Fastest:	0.0069 secs
      Average:	0.1257 secs
      Requests/sec:	347.2561
    
      Total data:	14.26 KiB
      Size/request:	73 B
      Size/sec:	24.75 KiB
    
    Response time histogram:
      0.007 [1]   |
      0.061 [142] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
      0.115 [0]   |
      0.169 [2]   |
      0.223 [5]   |■
      0.276 [4]   |
      0.330 [4]   |
      0.384 [5]   |■
      0.438 [4]   |
      0.492 [17]  |■■■
      0.546 [16]  |■■■
    
    Response time distribution:
      10.00% in 0.0103 secs
      25.00% in 0.0117 secs
      50.00% in 0.0147 secs
      75.00% in 0.2434 secs
      90.00% in 0.4841 secs
      95.00% in 0.5154 secs
      99.00% in 0.5386 secs
      99.90% in 0.5459 secs
      99.99% in 0.5459 secs
    
    
    Details (average, fastest, slowest):
      DNS+dialup:	0.0005 secs, 0.0001 secs, 0.0011 secs
      DNS-lookup:	0.0001 secs, 0.0000 secs, 0.0010 secs
    
    Status code distribution:
      [200] 200 responses

As a comparison this is how a similar apache-bench call looks like:


    $ ab -n 10 -c 10 http://localhost:3000/up
    This is ApacheBench, Version 2.3 <$Revision: 1843412 $>
    Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
    Licensed to The Apache Software Foundation, http://www.apache.org/
    
    Benchmarking localhost (be patient).....done
    
    
    Server Software:        
    Server Hostname:        localhost
    Server Port:            3000
    
    Document Path:          /up
    Document Length:        73 bytes
    
    Concurrency Level:      10
    Time taken for tests:   0.041 seconds
    Complete requests:      10
    Failed requests:        0
    Total transferred:      6500 bytes
    HTML transferred:       730 bytes
    Requests per second:    244.19 [#/sec] (mean)
    Time per request:       40.952 [ms] (mean)
    Time per request:       4.095 [ms] (mean, across all concurrent requests)
    Transfer rate:          155.00 [Kbytes/sec] received
    
    Connection Times (ms)
                  min  mean[+/-sd] median   max
    Connect:        0    0   0.2      1       1
    Processing:     7   22   8.8     24      34
    Waiting:        6   22   8.9     24      34
    Total:          7   23   8.8     25      34
    ERROR: The median and mean for the initial connection time are more than twice the standard
           deviation apart. These results are NOT reliable.
    
    Percentage of the requests served within a certain time (ms)
      50%     25
      66%     28
      75%     31
      80%     33
      90%     34
      95%     34
      98%     34
      99%     34
     100%     34 (longest request)


Oha does show a lot more information during running the load test and **most importantly the distribution of response codes**. These are essential in seeing improvements on "production" handling of concurrent loads. The goal should be to never have an exception occur within usual request timeout window (i.e. because the database is locked or similar).


## Step 2 - Baseline

The application gathers data which are received via HTTP POST. Each POST creates a number of measurements internally (depending on payload). To make load testing easier i temporarily added a param which generates random data mimicking what we actually receive / create. Thus each POST to this endpoint triggers ten database queries of which five insert or update data.

It's very convenient that `oha` makes it easy to run POST requests. With apache bench one would have to provide an additional POST payload file (which might be empty). Thus instead of using something like `ab -n 10 -c 1 -p empty_file_to_post http://localhost:3000/metrics?load_test=true` we us the below oha call


    ➜  samarbeid-stats git:(main) ✗ docker run -it --net=host oha:latest -c 1 -z 5s -m POST --latency-correction --disable-keepalive --redirect 0 http://localhost:3000/metrics\?load_test\=true
    Summary:
      Success rate:	100.00%
      Total:	5.0013 secs
      Slowest:	0.1182 secs
      Fastest:	0.0212 secs
      Average:	0.0290 secs
      Requests/sec:	34.5913
    
      Total data:	0 B
      Size/request:	0 B
      Size/sec:	0 B
    
    Response time histogram:
      0.021 [1]   |
      0.031 [154] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
      0.041 [10]  |■■
      0.050 [1]   |
      0.060 [1]   |
      0.070 [0]   |
      0.079 [0]   |
      0.089 [1]   |
      0.099 [2]   |
      0.108 [1]   |
      0.118 [1]   |
    
    Response time distribution:
      10.00% in 0.0231 secs
      25.00% in 0.0245 secs
      50.00% in 0.0266 secs
      75.00% in 0.0287 secs
      90.00% in 0.0306 secs
      95.00% in 0.0365 secs
      99.00% in 0.1023 secs
      99.90% in 0.1182 secs
      99.99% in 0.1182 secs
    
    
    Details (average, fastest, slowest):
      DNS+dialup:	0.0002 secs, 0.0001 secs, 0.0004 secs
      DNS-lookup:	0.0000 secs, 0.0000 secs, 0.0001 secs
    
    Status code distribution:
      [200] 172 responses
    
    Error distribution:
      [1] aborted due to deadline
    

Using a single thread to load test the application works fine. Response times are acceptable and all requests are successful (i assume the `[1] aborted due to deadline`is due to the 5 second timeout set for the oha load test). 

But as soon as the concurrency increases we suddenly see errors:



    ➜  samarbeid-stats git:(main) ✗ docker run -it --net=host oha:latest -c 3 -z 5s -m POST --latency-correction --disable-keepalive --redirect 0 http://localhost:3000/metrics\?load_test\=true
    Summary:
      Success rate:	100.00%
      Total:	5.0016 secs
      Slowest:	0.1647 secs
      Fastest:	0.0351 secs
      Average:	0.0889 secs
      Requests/sec:	34.1891
    
      Total data:	643.95 KiB
      Size/request:	3.83 KiB
      Size/sec:	128.75 KiB
    
    Response time histogram:
      0.035 [1]  |
      0.048 [0]  |
      0.061 [2]  |
      0.074 [17] |■■■■■■
      0.087 [82] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
      0.100 [45] |■■■■■■■■■■■■■■■■■
      0.113 [8]  |■■■
      0.126 [3]  |■
      0.139 [1]  |
      0.152 [0]  |
      0.165 [9]  |■■■
    
    Response time distribution:
      10.00% in 0.0732 secs
      25.00% in 0.0804 secs
      50.00% in 0.0848 secs
      75.00% in 0.0915 secs
      90.00% in 0.1061 secs
      95.00% in 0.1563 secs
      99.00% in 0.1624 secs
      99.90% in 0.1647 secs
      99.99% in 0.1647 secs
    
    
    Details (average, fastest, slowest):
      DNS+dialup:	0.0001 secs, 0.0001 secs, 0.0003 secs
      DNS-lookup:	0.0000 secs, 0.0000 secs, 0.0002 secs
    
    Status code distribution:
      [200] 153 responses
      [500] 15 responses
    
    Error distribution:
      [3] aborted due to deadline


and right enough the corresponding log output shows it's sqlites "fault":


    rails s | grep Exception
    ActiveRecord::StatementInvalid (SQLite3::BusyException: database is locked):
    ActiveRecord::StatementInvalid (SQLite3::BusyException: database is locked):


## Step 3 - Solution?

Stephen talked extensively in his talk about the reasons for this to happen and that it's actually mainly a problem
of changing a few sqlite settings to configure it for the type of concurrent load expected in web applications.

Luckely Stephen integrated all suggested fixes into a gem which applies those to your application. Thus a simple

    $ bundle add activerecord-enhancedsqlite3-adapter

is supposed to be enough to make sqlite a real production option.

But running a load test (after restarting rails server) does not show any improvement

    
    ➜  samarbeid-stats git:(main) ✗ docker run -it --net=host oha:latest -c 3 -z 5s -m POST --latency-correction --disable-keepalive --redirect 0 http://localhost:3000/metrics\?load_test\=true
    Summary:
      Success rate:	100.00%
      Total:	5.0008 secs
      Slowest:	0.1911 secs
      Fastest:	0.0524 secs
      Average:	0.0988 secs
      Requests/sec:	30.5952
    
      Total data:	601.02 KiB
      Size/request:	4.01 KiB
      Size/sec:	120.18 KiB
    
    Response time histogram:
      0.052 [1]  |
      0.066 [2]  |■
      0.080 [13] |■■■■■■■
      0.094 [56] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
      0.108 [50] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■
      0.122 [14] |■■■■■■■■
      0.136 [2]  |■
      0.150 [4]  |■■
      0.163 [2]  |■
      0.177 [3]  |■
      0.191 [3]  |■
    
    Response time distribution:
      10.00% in 0.0799 secs
      25.00% in 0.0888 secs
      50.00% in 0.0943 secs
      75.00% in 0.1024 secs
      90.00% in 0.1206 secs
      95.00% in 0.1522 secs
      99.00% in 0.1853 secs
      99.90% in 0.1911 secs
      99.99% in 0.1911 secs
    
    
    Details (average, fastest, slowest):
      DNS+dialup:	0.0002 secs, 0.0001 secs, 0.0004 secs
      DNS-lookup:	0.0000 secs, 0.0000 secs, 0.0003 secs
    
    Status code distribution:
      [200] 136 responses
      [500] 14 responses
    
    Error distribution:
      [3] aborted due to deadline


After some trial-and-error we discovered we already had changed one setting deviating from the default sqlite config (`retries: 1000 # retries instead of timeout (new rails 7.1 flag)`). Propably in hopeing this would avoid errors but we obviously never verified this...

After reverting `database.yml` to rails defaults


    default: &default
      adapter: sqlite3
      pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
      timeout: 5000


A subsequent load test no longer caused any `SQLite3::BusyException`


    ➜  samarbeid-stats git:(main) ✗ docker run -it --net=host oha:latest -c 3 -z 5s -m POST --latency-correction --disable-keepalive --redirect 0 http://localhost:3000/metrics\?load_test\=true
    Summary:
      Success rate:	100.00%
      Total:	5.0006 secs
      Slowest:	0.2517 secs
      Fastest:	0.0418 secs
      Average:	0.1042 secs
      Requests/sec:	29.1964
    
      Total data:	0 B
      Size/request:	0 B
      Size/sec:	0 B
    
    Response time histogram:
      0.042 [1]  |
      0.063 [2]  |
      0.084 [10] |■■■
      0.105 [82] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
      0.126 [32] |■■■■■■■■■■■■
      0.147 [8]  |■■■
      0.168 [2]  |
      0.189 [3]  |■
      0.210 [1]  |
      0.231 [1]  |
      0.252 [1]  |
    
    Response time distribution:
      10.00% in 0.0841 secs
      25.00% in 0.0913 secs
      50.00% in 0.0983 secs
      75.00% in 0.1133 secs
      90.00% in 0.1272 secs
      95.00% in 0.1603 secs
      99.00% in 0.2142 secs
      99.90% in 0.2517 secs
      99.99% in 0.2517 secs
    
    
    Details (average, fastest, slowest):
      DNS+dialup:	0.0001 secs, 0.0001 secs, 0.0005 secs
      DNS-lookup:	0.0000 secs, 0.0000 secs, 0.0003 secs
    
    Status code distribution:
      [200] 143 responses
    
    Error distribution:
      [3] aborted due to deadline


## Step 4 - Conclusion: Test driven development

It shows once more it's vital to measure Befor and After before applying optimizations. Or taken differently: Create a failing test first and verify your changes make it green.

Using oha for load testing instead of the traditional apache-bench provided more insightful data, including error code distribution, which were essential to ensured that our fixes were effective. It's certainly part of my go-to toolset.

Stepen's `activerecord-enhancedsqlite3-adapter` gem works wonders - really looking forward to his contributions being included into Rails (8?) by default!