---
title: "Scaling RL rollouts with Firecracker on Amazon EC2 metal"
url: "https://aws.amazon.com/blogs/compute/scaling-rl-rollouts-with-firecracker-on-amazon-ec2-metal/"
date: "2026-10-09"
author: "Karsten Ploesser"
feed_url: "https://aws.amazon.com/blogs/compute/feed/"
---
Reinforcement learning at scale runs thousands of isolated rollout sandboxes in parallel, and packing more of them onto each host lowers your cost per training step. This post shows how to tune Firecracker microVM density on Amazon EC2 metal, and introduces labsweep, a harness for mapping your own latency and density curve.
