---
title: VPC Lattice Part 1. Overview
date: "2023-04-03"
description: Making sense of the new application layer network managed service from AWS.
---
## Introduction

On the 31st of March 2023, VPC Lattice made General Availability. The new managed service is here to help us create logical application layer networks across VPCs and isolated AWS accounts. A user [guide is available](https://docs.aws.amazon.com/vpc-lattice/index.html), along with several examples. Throughout several posts, I will present an overview of the offering, some implementation details, and a deep dive into a potential reference architecture. This post is Part 1 of the series on Lattice and does not include an IaC follow-along.

## What Is Lattice and Why Do I Need It

Consider the use case: your company's landing zone on AWS consists of multiple isolated accounts managed from within a single AWS Organization. Each development team is assigned an account to deploy their services. To make service endpoints available for other teams in different AWS accounts to consume, you must ensure network connectivity between these accounts. Service-to-service connectivity is considered east-west traffic that does not leave your internal network boundary. To interconnect, you will consider several standard options:
1. To route through Transit Gateway - virtually connecting multiple VPCs through a central router and establishing omnidirectional connectivity between VPCs.
2. To publish each service as a PrivateLink endpoint. In such cases, a service consumer can establish unidirectional connectivity towards the service provider - an isolated development team's VPC. 

VPC Lattice addresses the pains associated with the second option. Let's explore our use case further. The development team in an isolated account manages an EKS cluster, exposing the applications internally via an AWS Load Balancer Controller. From the cloud perspective: we have a private ALB as an entry point to the workload. To publish this ALB for east-west consumption via PrivateLink, we have to build a "load balancer sandwich":
1. Create a Network Load Balancer.
2. Create a Network Load Balancer Target Group with a Target Type of Application Load Balancer.
3. Create a Network Load Balancer Listener.
4. Take care of SSL termination.
5. Create a PrivateLink endpoint service, pointing to the Network Load Balancer.
6. Add AWS principles to the Allow List of the endpoint service or otherwise manage ad-hoc connection requests. 

While the above steps can be easily scripted and packaged as a reusable IaC component, work is still required. In addition, cost becomes an issue with tens or hundreds of services, and Transit Gateway connectivity may become preferential.  

Lattice aims to provide a single-click tool to eliminate the steps. A value-added proposition of Lattice is the additional security control we can implement on top of our endpoint: auth policies. Auth policies allow restricting endpoint access to specific VPCs, accounts or even specific assumed IAM roles - which is excellent. 

Consider an application running in EKS in Account A that must perform an HTTP GET request to an application running in ECS in Account B. A Lattice service in Account B may have an auth policy specifying the exact IAM role assumed by the requester pod in account A through an OIDC-federated identity. We can lock it down further by only setting the allowed HTTP method GET. 

From the DevOps perspective, eliminating dependencies between dedicated cloud administrators and developers will improve delivery and backlog churn. But we still assume some level of netsec awareness by the development teams to manage the auth policies for their services. And we still need to establish a framework for governing the auth policies and enforcing the standards. 

## Limitations

Lattice's visible and significant downside is a one-to-one relationship between a VPC and a Service Network. This downside eliminates the possibility of building around a "shared services" scenario with network isolation. Consider the following situation:
1. Three VPCs in three separate AWS accounts belonging to an AWS Organization: SharedServices VPC, GreenTeam VPC, and BlueTeam VPC. 
2. SharedServices VPC hosts a SharedApp.
3. GreenTeam VPC and BlueTeam VPC host GreenApp and BlueApp, respectively.
4. GreenApp and BlueApp are allowed and must initiate HTTP requests to the SharedApp.
5. GreenApp and BlueApp are not allowed and must not initiate HTTP requests to each other. 
6. SharedApp is prohibited and must not initiate HTTP requests to BlueApp and GreenApp.

From a network isolation perspective, we would likely establish unidirectional connectivity with PrivateLink in the following pattern:
1. GreenApp (service consumer) -> SharedApp (service owner)
2. BlueApp (service consumer) -> SharedApp (service owner)

In the world of Lattice, GreenApp and BlueApp belong to two separate Service Networks, and they never need to meet. However, this would require the SharedApp to be simultaneously present on both Service Networks, which is impossible by design: you will get an ```ConflictException: VPC has already been associated to a service network``` exception if you try. 

The above limitation means we will require a single flat Service Network within the accounts' constellation. An alternative would be to revert to other networking tools at our disposal, e.g., PrivateLink and Transit Gateway. Another thing to remember is that Lattice objects work with AWS RAM and, as such, are regional. I.e. to establish inter-regional connectivity, we would still have to use Transit Gateway Inter-region Peering.

## Benefits

Regardless of the above limitations, using AWS Lattice is still beneficial in some situations. In principle, Lattice is a service discovery and networking solution that aims to simplify developers' lives by abstracting the steps required to publish an application over the local network boundary for east-west consumption across VPCs and AWS accounts. In the following posts, we will explore the implementation hands-on and look at a possible reference architecture to understand when it's best to use Lattice and when not to, possibly substituting it with other networking tools at our disposal. Stay tuned!
