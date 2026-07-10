---
title: Finding and Explaining Quiet AWS Cost Leaks in Non-Production Environments
slug: finding-and-explaining-quiet-aws-cost-leaks-in-non-production-environments
date_published: 1970-01-01T00:00:00.000Z
date_updated: 2026-03-18T04:51:56.000Z
draft: true
---

This started with a simple observation: non-production environments were costing more than expected. These environments aren't under constant load — most systems are used during working hours, with long idle periods at night. So the cost profile shouldn't resemble production. But it did.

I wasn't trying to build a perfect cost model or some "smart optimizer." I just wanted to answer a practical question: **where is money leaking quietly?**

So I ran a cost analysis across several non-prod AWS accounts, focusing on EC2, RDS, EBS, and related resources. The approach was straightforward. I pulled inventory across services using AWS APIs, flattened everything into CSVs, then enriched it with usage metrics where needed. For RDS, I included CloudWatch data like `CPUUtilization` and `FreeableMemory`, averaged over a few days to avoid short-term spikes. After that, I mapped everything to rough monthly costs — not exact billing numbers, just enough to understand the impact.

The script itself is intentionally simple. It flags things that look off, which is usually enough to start a conversation. No ML, no dashboards, just a CSV and some common sense.

---

## RDS: Oversized by Habit

RDS was the first thing that stood out. Several instances were running at consistently low CPU usage — often below 5% — while still using relatively large instance classes. Memory usage didn't justify the size either.

The script flagged these by combining low CPU with instance size. Then I simulated moving one size down. In many cases, that alone reduced the cost by roughly half. For smaller workloads, switching from x86 to ARM (Graviton) added another ~10% on top.

This pattern showed up across multiple instances that were likely sized once and never revisited. I get it — when you're setting up a staging environment, you just pick whatever works and move on. Nobody goes back to check if it's still the right size six months later.

Estimated savings: **~$660/month.**

---

## RDS: Running While Idle

The next issue was about runtime. Non-prod databases were running 24/7 even though they're mostly unused outside working hours. Nobody's running queries at 2 AM on a staging database.

I simulated a simple schedule: stop at midnight, start again in the morning. Used a proportional estimate — `(hours_off / 24) * monthly_cost`. Even with a conservative window, the savings were noticeable.

Estimated savings: **~$190/month.** This one depends on team usage patterns, so it needs validation before pulling the trigger. But honestly, if nobody complains about the database being down at 3 AM, you have your answer.

---

## EBS: Unattached Volumes

There were volumes in an "available" state — meaning not attached to anything. Just sitting there, costing money.

I filtered out recently created ones to reduce false positives and focused on older volumes. Individually small, but combined they add up. The main challenge isn't technical — it's making sure nobody still needs them. In a team where ownership is fuzzy, "just delete it" is rarely the right first move.

Estimated savings: **~$90/month.**

---

## Load Balancers: No Active Targets

Some ALBs had no registered targets. Detected this by checking for empty target groups.

Not all are safe to delete — some are tied to container setups where targets come and go dynamically. So this needs validation before acting. But if an ALB has been target-less for weeks, it's probably just forgotten infrastructure.

Estimated savings: **up to ~$70/month.**

---

## EC2: Stopped, But Still Costing

Stopped EC2 instances still incur EBS costs. One case had a relatively large volume sitting idle — the instance was stopped, but the disk was still billed.

Estimated savings: **~$20/month.** Small but easy to miss. The kind of thing that quietly leaks for months without anyone noticing.

---

## EBS Optimization: gp2 to gp3

Some volumes were still using gp2. Compared pricing with gp3 without needing any usage analysis — gp3 is cheaper and offers better baseline performance. No downside, low effort.

Estimated savings: **~$15/month.**

---

## Snapshots: Gradual Growth

Old EBS and RDS snapshots were flagged based on age. These don't spike costs, but they accumulate over time like dust. A snapshot from a test run six months ago? Probably safe to delete.

Estimated savings: **~$10/month.** Not urgent, but worth cleaning.

---

## CloudWatch Logs: No Retention

This one surprised me. Most log groups had no retention policy — meaning logs are stored indefinitely. Some had grown quite large without visibility. Setting retention to even 30 days would significantly reduce this.

Estimated savings: **~$100+/month** depending on the policy. The kind of cost that's invisible until you look.

---

## What Didn't Show Up

Elastic IPs were clean. No unattached resources there. At least one thing went right.

---

## Total Impact

Conservatively, total potential savings are around **~$1,100/month**. For non-production environments, that's meaningful.

But here's the thing: **none of these are dramatic fixes.** There's no single "aha" moment where you flip a switch and save a thousand dollars. It's a collection of small inefficiencies — oversized instances, systems running while idle, unused storage, defaults never revisited. Individually easy to ignore, but together they form a steady cost baseline.

The script didn't try to be smart. It just highlighted what didn't look right, and that's usually enough.

---

## Implementation Status

All findings are still in the observation and proposal phase. None have been implemented yet due to validation and coordination needs. Some resources may still be in use without clear ownership, and usage patterns need confirmation before applying changes.

This analysis is meant to highlight potential savings and start discussions, not enforce changes. Honestly, the hardest part of cost optimization isn't finding the leaks — it's convincing people that the thing they forgot about six months ago is safe to delete.