---
title: "Bring your own client certificate for backend mTLS in Amazon API Gateway"
url: "https://aws.amazon.com/blogs/compute/bring-your-own-client-certificate-for-backend-mtls-in-amazon-api-gateway/"
date: "2026-09-08"
author: "Biswanath Mukherjee"
feed_url: "https://aws.amazon.com/blogs/compute/feed/"
---
Enterprises that use Amazon API Gateway often want to bring their own client certificate for backend mutual TLS (mTLS) authentication. With API Gateway, you can now use a third-party or AWS Private CA-issued client certificate for the outbound mTLS handshake. In this post, you build a REST API with an outbound mTLS connection to an Amazon ECS backend.
