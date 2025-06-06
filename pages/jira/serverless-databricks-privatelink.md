---
date: 2022-09-01 1:00:00
title: Serverless Databricks with PrivateLink
published: True
tags:
  - terraform
  - aws
  - spacelift
---

# Introduction

We want to query some databases in our AWS account from our serverless
Databricks compute... The issue is that native PrivateLink support isn't
available for serverless Databricks and the feature they have in preview
requires some special setup.

## Resources

We will do as much as possible with Terraform.
Let's catalog the resources we'll need right away...

1. `aws_vpc_endpoint_service` - this is the PrivateLink endpoint service that will be used to connect to the Databricks workspace
2. `aws_lb` - a network load balancer
3. `aws_security_group` - security group for the load balancer
4. `aws_lb_target_group` - this is basically the groupings of IPs of the database endpoints we eventually want to be querying
5. `aws_lb_target_group_attachment` - an explicit resource for attaching the target groups to a load balancer
6. `aws_lb_listener` - a listener listens for traffic on a port and takes action, in our case it'll just forward the traffic
7. `aws_security_group_rule` - need to set ingress/egress rules for the load balancer to allow traffic to flow

One special thing we'll need, since we want to specify our endopints by DNS name, is a way to resolve that DNS to a set of IPs because target groups for NLBs in AWS cannot take VPC endpoints as their targets, it has to be IP addresses, so in terraform we will use hashicorp's `dns` provider for this

```terraform
terraform {
  required_providers {
    dns = {
      source = "hashicorp/dns"
    }
  }
}

data "dns_a_record_set" "target_lookup" {
  for_each = local.target_configs
  host     = each.key
}
```

## Limitations

There are some primary limitations to consider as we step through the plan...

1. Databricks limits workspaces to 1 Network Connectivity Configuration (NCC)
2. Any given NCC is limited to 3 PrivateLink endpoint configurations

# The Plans

1. Each database makes its own PL endpoint

- immediately we run into limitations where a workspace would be limited to access 3 databases at any given time

2. Provision 1 PrivateLink endpoint per account and route to every database in some clever way

- what if we want to query from databases across more than 3 accounts?

3. Provision 1 PrivateLink endpoint in a primary account, and then cleverly route traffic relying on Transit Gateway to permit cross-account/VPC traffic

## 3's the Winner

In order to meet this need/ask in a general way, option 3 is the only way to go

# Implementation

## Requirements First

We basically want a subnet router fronted by a PrivateLink endpoint.

So given a database endpoint and port we want to configure networking through the PrivateLink endpoint... This diagram helps it make sense I think

![cross account privatelink router](https://dropper.wayl.one/api/file/5330675c-bdb6-41c2-a021-4871b3e4f809.png)

## Details of Implementation

As this is more of a report rather than live-tweeting my experience I will spare every single issue but at the end I'll list some things I learned along the way

---

## Nuggets

- Coding with AI is helpful if you use the tool well, I tried to use it for generating terraform boilerplate in this project, and a few times I let it go off the rails and ultimately I had to backup... nothing super new here, but another confirmation that AI is tricky to work with if you want to produce valuable work
- PrivateLink is a very interesting piece of tech... I thought it was more like Wireguard but it's not P2P, it's more like a private tunnel that you can do with on either end what you want. In fact, this implementation turned our endpoint and NLB basically into a subnet router
