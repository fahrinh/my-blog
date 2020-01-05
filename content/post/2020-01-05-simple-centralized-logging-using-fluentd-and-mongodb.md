---
title: "Simple Centralized Logging Using FluentD and MongoDB"
date: 2020-01-05T09:52:58+07:00
draft: true
---

In microservices system, centralized logging is one of crucial things to consider at first place in system setup. We don't want to `ssh` to each server to `grep` or `tail` log files. That is cumbersome.

ELK (Elastic-Logstash-Kibana) is a popular logging stack right now. It is complete and full-featured. More importantly, it can be setup on-promise (self-hosted on your own server). I personally avoid cloud solutions to save budget.

So, what is downside of ELK ?

<!--more-->

It is heavy. ELK is mostly Java-based. Java is heavy. Yes, it is performant, yet still heavy in term of RAM usage.

Then, what is other viable alternatives ?

Firstly, we need to define ~~the~~ **my** requirements:

- It is self hosted. As I said, to save budget.
- It has not-so-high server requirements. The cost per GB for RAM is still high. Remember, I need to save budget.
- It provides basic log search functionality. I can query and search log based on logging level (`INFO`, `ERROR`, etc).
- It is easy to setup and maintenance.
- High availability is optional. I can tolerate if unfortunately the logging server is down or network has a trouble causing log data not transfered to logging server.

# FluentD + MongoDB

TODO:

# Simple Centralized Logging  

![Simple Centralized Logging Setup](http://www.plantuml.com/plantuml/proxy?src=https://raw.githubusercontent.com/fahrinh/my-blog/master/diagram/centralized-logging-setup.plantuml)


## Our Main Application

This is a sample application to produces logs. I use Go and [Zap](https://github.com/uber-go/zap)  as logging library. I recommend JSON as log format because as structured data format, it is easy to be parsed and be processed by log agent.



## FluentD Setup

## MongoDB Setup
