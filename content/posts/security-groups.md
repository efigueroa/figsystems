---
title: "Formatting AWS Security Groups for a VMware Migration"
date: "2025-02-05"
tags:
  - terraform
  - aws
  - migration
categories:
  - selfhosted
---

# The Problem

At work we're in the middle of a large lift and shift migration from VMware to AWS (for the same reason everyone is). Hundreds of servers across multiple departments, moved in waves.

The firewall rules for these servers come from everywhere. Palo Alto firewalls, host-based firewalls, department-specific switches, department-specific IT teams, random appliances that predate much of the current staff. Years of accumulated rules from multiple sources, and now they all need to become AWS security groups.

I needed to figure out how to format these rules in Terraform so that:
1. Coworkers completely new to IaC could read them
2. I could maintain them without losing my mind as rule counts climbed
3. PRs were reviewable

This is how the format evolved over three iterations.

# Iteration 1: Inline Rules

The most straightforward way to write a security group. Everything in one block.

```hcl
resource "aws_security_group" "web_server" {
  name        = "web-server"
  description = "SG for web-server"
  vpc_id      = var.vpc_id

  ingress {
    description = "HTTPS from campus"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/24"]
  }

  ingress {
    description = "SSH from admin subnet"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.100.0.0/24"]
  }

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

This works fine for a server with 3-4 rules and is the first example you usually come across if you search for "ec2 firewalls". It's easy to read and easy to explain to someone who's never seen Terraform before.

The problem is that any change to any inline rule forces Terraform to evaluate the entire security group. Add a CIDR to one ingress block and the plan output gets noisy. It also doesn't play well with `for_each` if you want to loop over CIDRs for a single port.

# Iteration 2: Separate Rule Resources

Breaking the rules out into their own resources using `aws_vpc_security_group_ingress_rule` and `aws_vpc_security_group_egress_rule`.

```hcl
resource "aws_security_group" "web_server" {
  description = "SG for web-server"
  vpc_id      = var.vpc_id

  tags = {
    Name   = "web-server"
    Source = "Palo Alto Firewall"
  }
}

# Egress
resource "aws_vpc_security_group_egress_rule" "web_server_allow_all_outbound" {
  security_group_id = aws_security_group.web_server.id
  ip_protocol       = "-1"
  cidr_ipv4         = "0.0.0.0/0"

  tags = {
    Name = "allow-all-outbound"
  }
}

# HTTPS from campus
resource "aws_vpc_security_group_ingress_rule" "web_server_https_443" {
  for_each          = var.https_443_cidrs
  security_group_id = aws_security_group.web_server.id
  cidr_ipv4         = each.key
  description       = each.value
  ip_protocol       = "tcp"
  from_port         = 443
  to_port           = 443

  tags = {
    Name = "HTTPS-443-${replace(each.key, "/", "-")}"
    Rule = "tcp-443"
  }
}

# SSH from admin subnet
resource "aws_vpc_security_group_ingress_rule" "web_server_ssh_22" {
  for_each          = var.ssh_22_cidrs
  security_group_id = aws_security_group.web_server.id
  cidr_ipv4         = each.key
  description       = each.value
  ip_protocol       = "tcp"
  from_port         = 22
  to_port           = 22

  tags = {
    Name = "SSH-22-${replace(each.key, "/", "-")}"
    Rule = "tcp-22"
  }
}
```

With variables like:

```hcl
variable "https_443_cidrs" {
  type = map(string)
  default = {
    "10.0.0.0/24"   = "Campus network"
    "10.100.0.0/24" = "Admin subnet"
  }
}

variable "ssh_22_cidrs" {
  type = map(string)
  default = {
    "10.100.0.0/24" = "Admin subnet"
  }
}
```

This is better. Each rule is its own resource so Terraform plans are cleaner. Adding a CIDR to a port only shows that one rule changing. The `for_each` on a map of CIDR-to-description means you can see at a glance what each IP range is for.

I used this format for the 2nd wave. It worked. But by the next few waves we were moving more servers per wave and each server had its own set of variables. The variable files were getting long and hard to cross-reference with the rules.

Everything was also moved into a `$WORKSPACE/modules/security-groups/` directory to keep it organized. One file per server's rules, one file per server's variables.

# Iteration 3: Locals with Structured Data

By the time we were moving double digit servers per wave, the variable-per-port approach was getting hard to maintain. Too many variable files, too much scrolling back and forth to understand what a server's rules actually looked like.

I switched to using `locals` with a structured list. All the rules for a server live in one block. Each entry defines the port, protocol, and every CIDR that needs access on that port.

```hcl
locals {
  web_server_ports = [
    # HTTPS
    {
      protocol = "tcp"
      from     = 443
      to       = 443
      name     = "https-443"
      cidrs = {
        "10.0.0.0/24"   = "Campus network"
        "10.100.0.0/24" = "Admin subnet"
      }
    },
    # SSH
    {
      protocol = "tcp"
      from     = 22
      to       = 22
      name     = "ssh-22"
      cidrs = {
        "10.100.0.0/24" = "Admin subnet"
      }
    },
    # RDP
    {
      protocol = "tcp"
      from     = 3389
      to       = 3389
      name     = "rdp-3389"
      cidrs = {
        "10.100.0.0/24" = "Admin subnet"
      }
    },
    # HTTP
    {
      protocol = "tcp"
      from     = 80
      to       = 80
      name     = "http-80"
      cidrs = {
        "10.0.0.0/24" = "Campus network"
      }
    },
  ]

  # Flatten into individual rules
  web_server_rules = flatten([
    for port_config in local.web_server_ports : [
      for cidr, description in port_config.cidrs : {
        key         = "${port_config.name}-${replace(cidr, "/", "-")}"
        protocol    = port_config.protocol
        from_port   = port_config.from
        to_port     = port_config.to
        cidr        = cidr
        description = description
        rule_name   = port_config.name
      }
    ]
  ])

  # How many rules total
  web_server_total_rule_count = length(local.web_server_rules)

  # How many SGs needed (AWS has a rules-per-SG limit)
  web_server_sg_count = max(1, ceil(local.web_server_total_rule_count / var.max_rules_per_sg))

  # Chunk rules across SGs
  web_server_rules_chunked = {
    for sg_index in range(local.web_server_sg_count) : sg_index => [
      for rule_index in range(
        sg_index * var.max_rules_per_sg,
        min((sg_index + 1) * var.max_rules_per_sg, local.web_server_total_rule_count)
      ) : local.web_server_rules[rule_index]
    ]
  }
}
```

The security group itself handles overflow automatically. If a server has more rules than AWS allows per SG, it creates additional SGs and distributes the rules across them. Neither I nor anyone in my team had to count rules to make sure they were split across security groups evenly. It all gets generated dynamically. 

```hcl
# Primary SG
resource "aws_security_group" "web_server" {
  name        = "web-server"
  description = "SG for web-server"
  vpc_id      = var.vpc_id

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name = "web-server"
  }
}

# Overflow SGs (created only if needed)
resource "aws_security_group" "web_server_overflow" {
  for_each = { for idx in range(1, local.web_server_sg_count) : idx => idx }

  name        = "web-server-overflow-${each.value}"
  description = "SG for web-server (Overflow ${each.value})"
  vpc_id      = var.vpc_id

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name = "web-server-overflow-${each.value}"
  }
}

# Egress (primary SG only)
resource "aws_vpc_security_group_egress_rule" "web_server_allow_all_outbound" {
  security_group_id = aws_security_group.web_server.id
  ip_protocol       = "-1"
  cidr_ipv4         = "0.0.0.0/0"

  tags = {
    Name = "allow-all-outbound"
  }
}

# Ingress for primary SG
resource "aws_vpc_security_group_ingress_rule" "web_server_ingress" {
  for_each = {
    for rule in local.web_server_rules_chunked[0] :
    rule.key => rule
  }

  security_group_id = aws_security_group.web_server.id
  cidr_ipv4         = each.value.cidr
  description       = each.value.description
  ip_protocol       = each.value.protocol
  from_port         = each.value.protocol == "-1" ? null : each.value.from_port
  to_port           = each.value.protocol == "-1" ? null : each.value.to_port

  tags = {
    Name = each.value.key
    Rule = each.value.rule_name
  }
}

# Ingress for overflow SGs
resource "aws_vpc_security_group_ingress_rule" "web_server_overflow_ingress" {
  for_each = merge([
    for sg_index, sg in aws_security_group.web_server_overflow : {
      for rule in local.web_server_rules_chunked[sg_index] :
      "${sg_index}-${rule.key}" => {
        sg_id       = sg.id
        cidr        = rule.cidr
        description = rule.description
        protocol    = rule.protocol
        from_port   = rule.from_port
        to_port     = rule.to_port
        key         = rule.key
        rule_name   = rule.rule_name
      }
    }
  ]...)

  security_group_id = each.value.sg_id
  cidr_ipv4         = each.value.cidr
  description       = each.value.description
  ip_protocol       = each.value.protocol
  from_port         = each.value.protocol == "-1" ? null : each.value.from_port
  to_port           = each.value.protocol == "-1" ? null : each.value.to_port

  tags = {
    Name = each.value.key
    Rule = each.value.rule_name
  }
}
```

Adding a new server means copying the template, doing a find-and-replace on the server name, and filling in the `ports` list. The SG resource, egress, overflow, and ingress logic are all identical across servers. The only thing that changes is the data in `locals`.

The big win for PR reviews is that the `ports` local reads like a table. You can look at it and immediately see what ports are open and to whom without having to mentally reconstruct it from scattered variable files.


# Standard Security Groups

While all the above handles per-server rules, we noticed early on that a lot of rules were the same across every server. RDP from the admin subnet, SSH from the admin subnet, ICMP from campus, etc. Every single server had these and we were duplicating them everywhere.

So we created a separate shared module: `$ROOT_OF_MONOREPO/modules/standard-securitygroups`. It only takes a `vpc_id` as input and creates a set of reusable security groups that any server can reference.

It does stuff like create our 3 admin groups:

- **default_admin** — ICMP and monitoring/backup access. No remote access.
- **linux_admin** - SSH mostly
- **windows_admin** - All the lovely SCCM/WSUS/SMB cruft from admin networks.

The key difference from per-server groups is that it uses managed prefix lists to centralize the IP ranges. Instead of hardcoding CIDRs in every rule, the rules reference a prefix list.

```hcl
resource "aws_ec2_managed_prefix_list" "linux_admin_access" {
  name           = "server-admin-access"
  address_family = "IPv4"
  max_entries    = 5

  entry {
    cidr        = "10.0.0.0/24"
    description = "Dept A linux Admin"
  }

  entry {
    cidr        = "10.100.0.0/24"
    description = "Dept B linux Admin"
  }
}
```

Then the rules reference the prefix list instead of individual CIDRs:

```hcl
resource "aws_vpc_security_group_ingress_rule" "linux_admin_ssh" {
  security_group_id = aws_security_group.linux_admin.id
  prefix_list_id    = aws_ec2_managed_prefix_list.server_admin_access.id
  ip_protocol       = "tcp"
  from_port         = 22
  to_port           = 22

  tags = {
    Name = "SSH-22-admin-access"
  }
}
```

When a new admin subnet needs access, you add one entry to the prefix list and every security group that references it picks it up. No touching individual server rules.

A server ends up with its per-server SG for application-specific rules and one or more standard SGs for the common stuff:

```hcl
vpc_security_group_ids = [
  module.security_groups.web_server_sg_id,
  module.standard_securitygroups.windows_admin_security_group_id
]
```

This keeps the per-server rule files focused on what's actually unique to that server.

# What's Next

The standard module handles the baseline admin access that every server gets. The next step is creating standard service-level and department-service-level SGs.

A generic `db-sg` would cover common database ports that most database servers need. But a `math-db-sg` would layer on department-specific rules for the math department's network ranges, their specific application servers, and their particular inter-database communication patterns. Same idea for web servers, app servers, etc.

The goal is to get to a point where standing up a new server means picking from a menu of standard SGs rather than writing rules from scratch every time.

# What I'd Do Differently

Not much honestly. The progression made sense given the constraints. We didn't know how many servers we'd be moving per wave at the start and the format evolved as the workload scaled. The template approach with find-and-replace is simple enough that even the folks brand new to Terraform are following along.
