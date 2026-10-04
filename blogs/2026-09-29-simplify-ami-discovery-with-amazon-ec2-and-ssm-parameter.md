---
title: "Simplify AMI discovery with Amazon EC2 and SSM Parameter Store"
url: "https://aws.amazon.com/blogs/compute/simplify-ami-discovery-with-amazon-ec2-and-ssm-parameter-store/"
date: "2026-09-29"
author: "Ashwani Tyagi"
feed_url: "https://aws.amazon.com/blogs/compute/feed/"
---
Managing Amazon EC2 AMIs at scale means constantly mapping AMI IDs to their AWS Systems Manager parameter paths by hand. A recent enhancement to the DescribeImages API returns the associated SSM parameter directly. This post shows how to use it across the AWS CLI, AWS CloudFormation, Terraform, and Auto Scaling launch templates.
