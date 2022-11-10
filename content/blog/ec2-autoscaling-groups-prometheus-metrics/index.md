---
title: AWS EC2 Auto Scaling, Target Tracking Policies and Prometheus Exporters
date: "2022-11-10"
description: Making EC2 autoscaling decisions based on metrics emitted by various Prometheus exporters.
---
## Introduction

Quite a few workloads may still require deployment to virtual machines instead of container orchestration platforms, yet still, have to be intelligently autoscaled. For example, a compound workload running on a stateless EC2 instance in an EC2 Autoscaling Group may emit various Prometheus-compatible metrics via Prometheus exporters. A simplified model for such a use case could be an Ubuntu instance running a node-exporter, exposing OS metrics, and a workload application delivering its own set of Prometheus-compatible metrics.

This post will illustrate how to configure an EC2 Autoscaling Group to use these metrics within Target Tracking Scaling Policies with the help of the AWS CloudWatch Agent.

Pulumi IaC will help us bring up our infrastructure on the AWS Cloud. Check out [pulumi.com](https://pulumi.com) if you still need to become familiar with it. You can deploy this demo stack using the Pulumi button below.

## What We Are Going To Build

We will create a pair of identical EC2 instances running in an EC2 Autoscaling Group, each running a demo web workload and exposing node-exporter and web-server metrics. Next, AWS CloudWatch Agent will scrape the metrics and publish them to CloudWatch. Finally, a Target Tracking Scaling Policy will govern autoscaling decisions based on a customized metric specification.

To start building, we are going to need to meet the prerequisites:
1. An AWS account with a named AWS CLI profile configured.
2. A working Pulumi environment.

> Work In Progress