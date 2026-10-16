---
title: "Reducing cross-AZ latency with the AWS Lambda metadata endpoint"
url: "https://aws.amazon.com/blogs/compute/reducing-cross-az-latency-with-the-aws-lambda-metadata-endpoint/"
date: "2026-10-09"
author: "Daniel Abib"
feed_url: "https://aws.amazon.com/blogs/compute/feed/"
---
AWS Lambda now exposes Availability Zone metadata through an endpoint in the execution environment, so your function can discover its AZ ID with an HTTP request. Use it to prefer same-AZ endpoints for services like ElastiCache, Amazon RDS, and MemoryDB, reduce cross-AZ latency and data transfer costs, and build AZ-aware resilience patterns.
