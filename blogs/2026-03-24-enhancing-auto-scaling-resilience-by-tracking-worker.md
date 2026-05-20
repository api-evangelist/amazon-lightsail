---
title: "Enhancing auto scaling resilience by tracking worker utilization metrics"
url: "https://aws.amazon.com/blogs/compute/enhancing-auto-scaling-resilience-by-tracking-worker-utilization-metrics/"
date: "Tue, 24 Mar 2026 16:17:58 +0000"
author: "Brian Moore"
feed_url: "https://aws.amazon.com/blogs/compute/feed/"
---
A resilient auto scaling policy requires metrics that correlate with application utilization, which may not be tied to system resources. Traditionally, auto scaling policies track system resource such as CPU utilization. These metrics are easily available, but they only work when resource consumption correlates with worker capacity.
