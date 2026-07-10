---
title: Misreading Terraform Plan Led to Instance Destruction
slug: misreading-terraform-plan-led-to-instance-destruction
date_published: 2026-04-21T17:12:44.000Z
date_updated: 2026-04-21T17:12:44.000Z
excerpt: I made a small Terraform change and assumed it would be safe. The plan already showed that the instance would be replaced, but I did not fully process it. Within minutes, my server was gone. This write-up explains what happened and what I changed afterward.
---

This incident happened on Tencent Cloud and involved a CVM instance that I originally created manually for a PoC. Over time, that instance was reused for production without being rebuilt properly under Terraform. I later brought it into Terraform management, but I did not enable deletion protection, which became an important gap.

The problem started from a small configuration change. I updated `associated_ip_public` from `true` to `false`, assuming it would be applied in place. In reality, this attribute forces resource replacement. Terraform already showed this clearly in the plan output with `destroy and then create replacement` and a note indicating that the change forces replacement. I saw the plan, but I did not fully process what it meant. I focused too much on the summary and not enough on the actual behavior of the resource.

When I ran `terraform apply`, the existing CVM instance was destroyed and replaced with a new one. Since the instance was directly exposed using an Elastic IP and not behind any load balancer, the service immediately became unavailable. The downtime lasted around 15 minutes. There was no data loss because the system was still in early production use, but the situation could have been much worse.

To recover, I recreated the system disk from the latest snapshot, converted it into a custom image, and launched a new CVM instance from that image. I then reallocated an Elastic IP using the same public IP address, which allowed me to avoid any DNS changes because the A record was already pointing to that IP.

Looking back, the root cause was not Terraform itself but how I handled the plan. I assumed the change was safe, while Terraform clearly indicated a full replacement. I also did not have deletion protection enabled, so there was nothing preventing the destroy action from going through.

After this incident, I enabled deletion protection on production resources and increased snapshot frequency to reduce recovery gaps. I also became much stricter in reviewing `terraform plan`, especially when I see indicators that a resource will be replaced. For production changes, I strongly recommend having a second person review the plan together, since it is easy to miss critical details when reviewing alone.
