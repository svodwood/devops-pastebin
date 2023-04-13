---
title: VPC Lattice Part 2. Connecting Applications to Shared Services
date: "2023-04-14"
description: Utilizing VPC Peering to separate AWS VPC Lattice Networks while providing access to a shared HTTP service.
---
## Introduction

In Part 1 of the series, we looked at some of the benefits and drawbacks of VPC Lattice and theorized on possible use cases where this managed service might be helpful. We also discussed a "shared service" scenario, where some individual applications may desire access to a shared service while belonging to entirely separate networks. This pattern might present itself with a commonly shared HTTP endpoint - for example, for token generation and validation, tied to a shared encryption infrastructure. While not straightforward, utilizing VPC Lattice to support this topology is possible. Let's now dive deeper and construct a functional multi-account and multi-VPC mini-landing zone - utilizing VPC Lattice as an application layer network.

Pulumi IaC will help us bring up our infrastructure on the AWS Cloud. Check out pulumi.com if you still need to become familiar with it. You can deploy these demo stacks using the Pulumi buttons below.

>Disclaimer #1: in the Pulumi stacks, we use two AWS providers: aws and aws-native, which need separate configurations, e.g., AWS CLI profile name and region. The latter is the only option to work with Lattice from Pulumi. I cannot currently recommend the aws-native provider for production use.

>Disclaimer #2: at the time of writing, some requests to VPC Lattice API return 429. AWS is still throttling requests to the service, although this may be related to your account status. Run "pulumi up" one more time if you experience this.

## What We Are Going To Build

Our focus today is on our hypothetical company's three microservices:
1. DirectoryService. A shared microservice.
2. BlueService. A standalone microservice.
3. GreenService. A standalone microservice.

Separate development teams own the BlueService and the GreenService, respectively - each team has an isolated AWS account where they develop their microservices. An independent team maintains the third application (DirectoryService) and controls access to the application. This application runs in the third isolated AWS account.

Our network isolation requirements define that the GreenService must not be in the same network as the BlueService. Both services, however, need to perform HTTP GET requests to the DirectoryService - using dedicated query strings.

Below is the high-level diagram of the landing zone topology and network isolation requirements:

![VPC Lattice Shared Service Topology](./LatticeSharedServiceDiagram.png)

To start building, we are going to need to meet the prerequisites:
1. Three separate AWS accounts belong to a single AWS Organization. Additionally, we must enable RAM sharing across the Organization.
2. Three configured CLI profiles - for all three accounts, respectively. Mind that the aws:profile and aws-native:profile providers must be configured explicitly and separately. They may, however, reuse the same credentials.
3. A working Pulumi environment.
4. Account IDs of all three AWS accounts.

## Pulumi Stacks and Deployment Procedures

Our lab repository consists of three directories, each corresponding to a separate Pulumi stack:
1. ./a-common-service
2. ./b-blue-service
3. ./c-green-service

A "common-service" stack contains the infrastructure for the DirectoryService application deployed to a separate AWS account. This specific account also owns the two independent VPC Lattice Service Networks: 
1. One is to control the communication with the GreenService.
2. Another is to control the communication with BlueService.

We share the Service Networks with the respective microservice accounts via AWS RAM. As such, we provision the stacks in the same order listed above. Once the "common-service" stack is up, note the two VPC Lattice Service Networks - we must use their ARNs in the blue and green stack configurations, respectively. For simplicity, we will not use stack imports here and will copy-paste the correct values of the "common-service" stack output. Also, make a note of the "shared-service-fqdn" output value. Then, copy-paste that into both "blue-service" and "green-service" stack inputs.

```python
import json
```

## Cleaning Up

Run "pulumi destroy" on all the stacks in reverse order. Start from "green-service", then "blue-service", then "common-service". Afterwards, delete CloudWatch Lambda log groups from all three accounts manually if needed. Due to AWS's throttling, you may need to manually delete some VPC Lattice objects and do a "pulumi refresh" instead, although there is usually no need to. If you experience throttling, running "pulumi destroy" several times resolves the issue.
