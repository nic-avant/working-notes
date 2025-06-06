---
cover: ''
date: 2022-09-01
datetime: 2022-09-01 00:00:00+00:00
description: We want to query some databases in our AWS account from our serverless
  We will do as much as possible with Terraform. aws_vpc_endpoint_service aws_lb aws_securi
edit_link: https://github.com/edit/main/pages/jira/serverless-databricks-privatelink.md
jinja: false
long_description: We want to query some databases in our AWS account from our serverless
  We will do as much as possible with Terraform. aws_vpc_endpoint_service aws_lb aws_security_group
  aws_lb_target_group aws_lb_target_group_attachment aws_lb_listener aws_security_g
now: 2025-06-06 18:34:08.208255
path: pages/jira/serverless-databricks-privatelink.md
published: true
slug: serverless-databricks-privatelink
status: draft
super_description: 'We want to query some databases in our AWS account from our serverless
  We will do as much as possible with Terraform. aws_vpc_endpoint_service aws_lb aws_security_group
  aws_lb_target_group aws_lb_target_group_attachment aws_lb_listener aws_security_group_rule
  One special thing we There are some primary limitations to consider as we step through
  the plan... Databricks limits workspaces to 1 Network Connectivity Configuration
  (NCC) Any given NCC is limited to 3 PrivateLink endpoint configurations '
tags:
- terraform
- aws
- spacelift
templateKey: ''
title: Serverless Databricks with PrivateLink
today: 2025-06-06
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

Users should be able to expose a database endpoint in any AWS account
sufficiently connected via Transit Gateway, to this PrivateLink and any of the
consumers on the other end of the PrivateLink relationship.

So given a database endpoint and port we want to configure networking through
the PrivateLink endpoint... This diagram helps it make sense I think

![cross account privatelink router](https://dropper.wayl.one/api/file/5330675c-bdb6-41c2-a021-4871b3e4f809.png)

## Details of Implementation

> As this is more of a report rather than live-tweeting my experience I will
> spare every single issue but at the end I'll list some things I learned along
> the way

I listed the resources above but let's do an overview of the key components of the module...

First let's see the variables we want:

```terraform
variable "vpc_id" {
  type        = string
  description = "The ID of the VPC where resources will be created"
}

variable "subnet_ids" {
  type        = list(string)
  description = "List of subnet IDs where the Network Load Balancer will be created"
}

variable "tags" {
  type        = map(string)
  description = "Tags to be applied to all resources"
  default     = {}
}

variable "service_name" {
  type    = string
  default = "privatelink-router"
}

variable "target_endpoints" {
  type = list(object({
    id   = string
    port = number
    name = string
  }))
  description = "List of target endpoints with hostname and port"
  default     = []
}

variable "service_config" {
  type = object({
    target_type = optional(string, "ip")
    protocol    = optional(string, "TCP")
    base_port   = optional(number, 10000)
    # NOTE: I wrote my module so that service_config.targets gets merged with
    #target_endpoints as we plan to supply target_endpoints in environment
    #specific var files
    targets = list(object({
      id          = string
      target_port = number
      protocol    = optional(string)
    }))
  })
  description = "Configuration for the service endpoints. Each target can specify its own target_port (destination port) and optional protocol override."

  default = { targets = [] }
}

variable "health_check_config" {
  type = object({
    port                = optional(string, "traffic-port")
    path                = optional(string)
    protocol            = optional(string)
    healthy_threshold   = optional(number, 3)
    unhealthy_threshold = optional(number, 3)
    interval            = optional(number, 30)
    timeout             = optional(number, 10)
  })
  description = "Configuration for target group health checks"
  default = {
    port                = "traffic-port"
    path                = null
    protocol            = null
    healthy_threshold   = 3
    unhealthy_threshold = 3
    interval            = 30
    timeout             = 10
  }
}

```

Now get the `locals` and `data` into your head:

```terraform
locals {
  nlb_name = "${var.service_name}-nlb"

  # Transform target_endpoints into service_config targets format
  service_targets = [
    for endpoint in var.target_endpoints : {
      id          = endpoint.id
      name        = endpoint.name
      target_port = endpoint.port
      protocol    = null # Using default protocol from service_config
    }
  ]

  # Combine the transformed targets with service_config
  combined_service_config = merge(var.service_config, {
    targets = concat(var.service_config.targets, local.service_targets)
  })

  # Base port for service endpoints
  base_port = 10000

  # Process target configurations
  target_configs = {
    for idx, target in local.combined_service_config.targets : target.id => {
      name        = try(target.name, replace(target.id, "/[^a-zA-Z0-9-]/", "-"))
      target_port = target.target_port
      source_port = local.base_port + idx
      protocol    = coalesce(target.protocol, local.combined_service_config.protocol, "TCP")
    }
  }

  # Resolve target IPs using DNS lookup
  resolved_ips = {
    for id, config in local.target_configs : id => distinct(
      data.dns_a_record_set.target_lookup[id].addrs
    )
  }

  # Security group rules
  security_group_rules = {
    ingress_rules = [
        {
          from_port   = local.base_port
          to_port     = max(local.base_port, local.base_port + length(local.target_configs) - 1)
          protocol    = "tcp"
          cidr_blocks = ["10.0.0.0/8"]
          description = "Allow inbound traffic to service endpoints"
        }
      ]
    egress_rules = [
        {
          from_port   = 0
          to_port     = 65535
          protocol    = "-1"
          cidr_blocks = ["10.0.0.0/8"]
          description = "Allow outbound traffic to internal network"
        }
      ]
}


# DNS Lookup for Target Group endpoints
data "dns_a_record_set" "target_lookup" {
  for_each = local.target_configs
  host     = each.key
}
```

And with those in mind we can take a look at the resources - with basic terraform familiarity I think it's not too much to take in

```terraform
# Security Group for NLB
resource "aws_security_group" "nlb_ingress" {
  name        = "${local.nlb_name}-ingress"
  description = "Security Group for NLB Ingress"
  vpc_id      = data.aws_vpc.selected.id
}

# Network Load Balancer
resource "aws_lb" "lb" {
  name               = local.nlb_name
  internal           = true
  load_balancer_type = "network"
  subnets            = var.subnet_ids
  security_groups    = [aws_security_group.nlb_ingress.id]
}

# Target groups for NLB
resource "aws_lb_target_group" "target" {
  for_each = local.target_configs

  name        = "${local.nlb_name}-tg-${each.value.name}"
  port        = each.value.target_port
  protocol    = each.value.protocol
  target_type = var.service_config.target_type
  vpc_id      = data.aws_vpc.selected.id
}

# Attach resolved IPs to target groups
resource "aws_lb_target_group_attachment" "tga" {
  for_each = {
    for pair in flatten([
      for target_id, ips in local.resolved_ips : [
        for ip in ips : {
          key       = "${target_id}-${ip}"
          target_id = target_id
          ip        = ip
        }
      ]
    ]) : pair.key => pair
  }

  target_group_arn = aws_lb_target_group.target[each.value.target_id].arn
  target_id        = each.value.ip
  port             = local.target_configs[each.value.target_id].target_port
}

# NLB Listeners
resource "aws_lb_listener" "listener" {
  for_each = local.target_configs

  load_balancer_arn = aws_lb.lb.arn
  port              = each.value.source_port
  protocol          = each.value.protocol

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.target[each.key].arn
  }
}

resource "aws_vpc_endpoint_service" "privatelink" {
  acceptance_required = true
  allowed_principals = ["arn:aws:iam::565502421330:role/private-connectivity-role-us-east-2"]
  network_load_balancer_arns = [aws_lb.lb.arn]
}

# Security Group Rules
resource "aws_security_group_rule" "nlb_ingress" {
  security_group_id = aws_security_group.nlb_ingress.id
  type              = "ingress"
  from_port         = local.security_group_rules.ingress_rules[0].from_port
  to_port           = local.security_group_rules.ingress_rules[0].to_port
  protocol          = local.security_group_rules.ingress_rules[0].protocol
  cidr_blocks       = local.security_group_rules.ingress_rules[0].cidr_blocks
  description       = local.security_group_rules.ingress_rules[0].description
}

resource "aws_security_group_rule" "nlb_egress" {
  security_group_id = aws_security_group.nlb_ingress.id
  type              = "egress"
  from_port         = local.security_group_rules.egress_rules[0].from_port
  to_port           = local.security_group_rules.egress_rules[0].to_port
  protocol          = local.security_group_rules.egress_rules[0].protocol
  cidr_blocks       = local.security_group_rules.egress_rules[0].cidr_blocks
  description       = local.security_group_rules.egress_rules[0].description
}
```

The PrivateLink endpoint is created at the VPC level, and then is supported by
a Network Load Balancer (NLB). The NLB is created with a security group that
has an ID which we'll need later. In the meantime, the NLB has associated
Listeners - they listen on specific ports and take action, like forwarding TCP
traffic to a Target Group. In this case, we construct the Target Groups for
each database endpoint a user wants to expose by performing a DNS lookup, and
then creating a Target Group Attachment for each of the resolved IPs for the
Target Group that we're creating for the given database.

So request comes into the PrivateLink endpoint on a specific port, the listener
on the NLB forwrads the traffic to the appropriate Target Group whose
attachments are the correct IPs of the database

The final thing on our end is to ensure that the security group rules for the
database endpoint are configured to allow traffic from the NLB - which we can
do by adding the security group id to any existing rules. We can do this in the
console or if the desired target endpoint is created with a terrform module we
can update the terraform module and even hard code the security group id of the
NLBa

# FIN

With that we now have a working PrivateLink endpoint that we can use to expose
any number of databases to Databricks Serverless without being hamstrung by the
limitations of 3 endpoints per NCC and 1 NCC per workspace!

# Nuggets

- Coding with AI is helpful if you use the tool well, I tried to use it for generating terraform boilerplate in this project, and a few times I let it go off the rails and ultimately I had to backup... nothing super new here, but another confirmation that AI is tricky to work with if you want to produce valuable work
- PrivateLink is a very interesting piece of tech... I thought it was more like Wireguard but it's not P2P, it's more like a private tunnel that you can do with on either end what you want. In fact, this implementation turned our endpoint and NLB basically into a subnet router
<div class='prevnext'>

    <style type='text/css'>

    :root {
      --prevnext-color-text: #eefbfe;
      --prevnext-color-angle: #e1bd00c9;
      --prevnext-subtitle-brightness: 3;
    }
    [data-theme="light"] {
      --prevnext-color-text: #1f2022;
      --prevnext-color-angle: #ffeb00;
      --prevnext-subtitle-brightness: 3;
    }
    [data-theme="dark"] {
      --prevnext-color-text: #eefbfe;
      --prevnext-color-angle: #e1bd00c9;
      --prevnext-subtitle-brightness: 3;
    }
    .prevnext {
      display: flex;
      flex-direction: row;
      justify-content: space-around;
      align-items: flex-start;
    }
    .prevnext a {
      display: flex;
      align-items: center;
      width: 100%;
      text-decoration: none;
    }
    a.next {
      justify-content: flex-end;
    }
    .prevnext a:hover {
      background: #00000006;
    }
    .prevnext-subtitle {
      color: var(--prevnext-color-text);
      filter: brightness(var(--prevnext-subtitle-brightness));
      font-size: .8rem;
    }
    .prevnext-title {
      color: var(--prevnext-color-text);
      font-size: 1rem;
    }
    .prevnext-text {
      max-width: 30vw;
    }
    </style>
    
    <a class='prev' href='/building-the-site'>
    

        <svg width="50px" height="50px" viewbox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
            <path d="M13.5 8.25L9.75 12L13.5 15.75" stroke="var(--prevnext-color-angle)" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"> </path>
        </svg>
        <div class='prevnext-text'>
            <p class='prevnext-subtitle'>prev</p>
            <p class='prevnext-title'>Building the Site</p>
        </div>
    </a>
    
    <a class='next' href='/'>
    
        <div class='prevnext-text'>
            <p class='prevnext-subtitle'>next</p>
            <p class='prevnext-title'>Markata Blog Starter</p>
        </div>
        <svg width="50px" height="50px" viewbox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
            <path d="M10.5 15.75L14.25 12L10.5 8.25" stroke="var(--prevnext-color-angle)" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"></path>
        </svg>
    </a>
  </div>