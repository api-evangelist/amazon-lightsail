---
title: "Adding custom domains to AWS Lambda MicroVMs with Application Load Balancer"
url: "https://aws.amazon.com/blogs/compute/adding-custom-domains-to-aws-lambda-microvms-with-application-load-balancer/"
date: "2026-09-22"
author: "Frank Scarfo"
feed_url: "https://aws.amazon.com/blogs/compute/feed/"
---
Many teams want to expose their AWS Lambda MicroVMs under a custom domain they own, and satisfy CORS for browser clients, without changing the application. This post shows how, using an Application Load Balancer that rewrites the Host header and forwards over AWS PrivateLink, deployed with the AWS CDK.
