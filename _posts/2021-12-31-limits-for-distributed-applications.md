---
layout: post
title:  "A Guide to Understanding Limits for Distributed Applications"
author: sam
categories: [ devops ]
tags: [red]
image: assets/images/budget.jpg
description: "A Guide to Understanding Limits for Distributed Application"
featured: true
hidden: false
toc: true
#rating: 4.5
---

Operating a service on the AWS platform can sometimes seem daunting. There seem to be so many moving parts. For example, regions, logs, hosts and more. In spite of the numerous resources available to help guide developers, limit management is often forgotten.

A common source of failure especially when building microservice based workloads is resource constraints. For example, when you reach the maximum number of APIs per region using App Sync or hit the maximum number of Apps you can create with the Amplify Console.

There are 3 key components to understand limit management

1. First, understand physical constraints to help you design a reliable system.
2. Second, identify the service limits of your dependencies.
3. Lastly, alert and report when you are about to hit a limit or hit a limit.

### Understand Physical Constraints

If you're running your workloads on [EC2 instances](https://aws.amazon.com/ec2/?nc2=h_m1), it is crucial that you monitor and frequently adjust hardware configuration based on the performance of your workload. [Amazon CloudWatch](https://aws.amazon.com/cloudwatch/?nc2=h_m1) can be used to set alarms to indicate when you are getting close to limits in Network IO, Provisioned IOPS, EBS etc. You can also set alarms for when you are approaching maximum capacity of auto scaling groups.

### Identify service limits of your dependencies

AWS does a great job of making information  available about service limits. You can find limits for most services here [service limits](https://docs.aws.amazon.com/general/latest/gr/aws_service_limits.html). These limits are tracked per account, so if you use multiple accounts, you need to know what the limits are in each account. Other limits may be based on your configuration. Limits are enforced per AWS Region and per AWS account. If you are planning to deploy into multiple regions or AWS accounts, then you should ensure that you increase limits in the regions and accounts that you using.

### Alert and report

AWS provides a list of some service limits via [AWS Trusted Advisor](https://aws.amazon.com/premiumsupport/technology/trusted-advisor/), and others are available from the AWS Management Console. The default service limits that are provided are available in the Service Limits documentation; you can contact AWS Support to provide your current limits for the services you are using if you have not tracked your limit increase requests.

<img src="https://nimacks.s3-us-west-2.amazonaws.com/img/limits.png" alt="drawing" width="765"/>

Ideally, limit tracking should be automated. You can store what your current service limits are in a persistent data store like Amazon DynamoDB. If you integrate your Configuration Management Database (CMDB) or ticketing system with AWS Support APIs, you can automate the tracking of limit increase requests and current limits.

In conclusion, you can save yourself a lot of headache by being intentional about how you monitor service limits. This will make your customers happy and your application or service very [reliable](https://wa.aws.amazon.com/wat.pillar.reliability.en.html). 
