---
title: "Introducing EC2 AMI Tag Sharing: Share EC2 tags across AWS accounts"
url: "https://aws.amazon.com/blogs/compute/introducing-ec2-ami-tag-sharing-share-ec2-tags-across-aws-accounts/"
date: "2026-10-05"
author: "Gena Gizzi"
feed_url: "https://aws.amazon.com/blogs/compute/feed/"
---
EC2 Tag Sharing lets AMI owners share tags with any account that has access to the image, using the ec2:SharedTag/ prefix. Updates propagate automatically with no custom replication workflows. This post walks through how it works, a golden AMI use case, and considerations for ABAC policies and account governance.
