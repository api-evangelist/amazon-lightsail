---
title: "Implementing customer managed keys for AWS Lambda durable functions with Terraform"
url: "https://aws.amazon.com/blogs/compute/implementing-customer-managed-keys-for-aws-lambda-durable-functions-with-terraform/"
date: "2026-09-29"
author: "Rajdeep Banerjee"
feed_url: "https://aws.amazon.com/blogs/compute/feed/"
---
Lambda durable functions checkpoint execution state to durable storage, and for regulated payment workloads that data is sensitive. This post shows how to configure a customer managed key in AWS KMS to encrypt durable execution data, define a least-privilege key policy, and verify encryption through AWS CloudTrail, all deployed with Terraform.
