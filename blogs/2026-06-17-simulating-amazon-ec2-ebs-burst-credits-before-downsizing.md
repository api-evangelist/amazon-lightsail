---
title: "Simulating Amazon EC2 EBS burst credits before downsizing an instance"
url: "https://aws.amazon.com/blogs/compute/simulating-amazon-ec2-ebs-burst-credits-before-downsizing-an-instance/"
date: "2026-06-17"
author: "Vineedh George"
feed_url: "https://aws.amazon.com/blogs/compute/feed/"
---
When downsizing an Amazon Elastic Compute Cloud (Amazon EC2) instance, teams often evaluate CPU and memory utilization but overlook the instance’s Amazon Elastic Block Store (Amazon EBS) performance limits for throughput and IOPS. Smaller Amazon EBS-optimized instance types have lower baselines and rely on burst credits to handle peaks. If your workload’s I/O pattern drains […]
