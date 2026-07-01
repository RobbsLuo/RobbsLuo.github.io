---
title: Introduction to Siege
date: 2016-10-24 14:46:29
tags:
  -  Server
  -  Siege
categories:
  -  Software
description: A brief introduction to the parameters of the load testing tool Siege
lang: en
---

## Introduction
  [Siege](https://github.com/JoeDog/siege 'SIege') is a load testing and benchmarking tool for Linux, designed for web developers to evaluate how well an application holds up under stress. It can be configured to perform concurrent access by multiple users against a web site, recording the response time of every request for each user, and repeating this under a given number of concurrent users. It supports multiple connections as well as GET and POST requests.

## Installation
 * Mac OSX
   ```bash
    brew install siege
   ```
 * Ubuntu
   ```bash
    aptitude install siege
   ```

## Parameters
```bash
-c:  Simulate N concurrent users accessing the target

-r:  Repeat the test run N times

-t:  Duration of the test run. The default unit is minutes, e.g. 5s (5 seconds) or 5 (5 minutes).
# -r and -t are generally not used together

-f:  The URL list for the workload

-i:  Randomly access URLs from the url.txt file specified by -f, in order to simulate real access patterns (randomness)

-b:  Run a stress test with no delay between requests (delay=0)

-A:  Specify the User-Agent for the requests
-H:  Specify a Header for the requests
-T:  Specify the Content-Type for the requests
```
 > ` siege -c 200 -r 100 http://www.google.com `
 > `siege  -c 200 -r 100 -f urls.txt`
 > `siege  -c 200 -r 100 -f urls.txt -i `
 > `siege -c 200 -r 100 -f urls.txt -i -b` delay=0, a more accurate stress test rather than a functional test
 > `siege -H "Content-Type:application/json" -c 200 -r 100 -f urls.txt -i -b`

## Result Explanation

  ```bash
  Transactions:		            20 hits            # Total number of tests performed
  Availability:		            100.00 %           # Success ratio
  Elapsed time:		            1.52 secs          # Total time elapsed
  Data transferred:	            0.80 MB            # Total data transferred
  Response time:		            0.14 secs          # Response time
  Transaction rate:	            13.16 trans/sec    # Requests processed per second
  Throughput:		            0.52 MB/sec        # Throughput rate
  Concurrency:		            1.82               # Highest concurrency
  Successful transactions:            20                 # Number of successful requests
  Failed transactions:                0                  # Number of failed requests
  Longest transaction:                0.51               # Longest time for a single transaction
  Shortest transaction:               0.05               # Shortest time for a single transaction
```

## Tips
  > When sending a POST request, the URL format is: `http://www.xxxx.com/ POST p1=v1&p2=v2`

  >  If the URL contains spaces or non-ASCII characters, it must be encoded first.
