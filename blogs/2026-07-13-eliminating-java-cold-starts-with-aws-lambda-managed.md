---
title: "Eliminating Java cold starts with AWS Lambda Managed Instances"
url: "https://aws.amazon.com/blogs/compute/eliminating-java-cold-starts-with-aws-lambda-managed-instances/"
date: "2026-07-13"
author: "Jay Colodner"
feed_url: "https://aws.amazon.com/blogs/compute/feed/"
---
A single cold start can push your Java Lambda function’s response time from milliseconds to seconds, enough to violate your p99 SLA, timeout a downstream service, and page your on-call. The Java Virtual Machine (JVM) performs best in long-running processes. Its Just-In-Time (JIT) compiler progressively optimizes code over thousands of invocations.
