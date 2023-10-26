---
title: Audit
subtitle: Is your database in good condition?

# Summary for listings and search engines
summary: Is your database in good condition? An audit ensures that your database is correctly configured, not only for reliability and performance, but also for costs.

# Date published
date: '2023-10-26T00:00:00Z'

# Date updated
lastmod: ''

# Is this an unpublished draft?
draft: false

# Show this page in the Featured widget?
featured: false
reading_time: false
show_date: false

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: ''
  focal_point: ''
  placement: 2
  preview_only: false

authors:
  - adrien

# tags:
#   - Audit
#
# categories:
#   - Audit
---

This is often the starting point for most of my interventions. Before working on specific problems, we need to make sure that the foundations are well established.

I always start with the lower layers, then work my way up to the applications:

## Linux configuration and tuning

I inspect storage (controller cards, raid, LVM...), Linux kernel configuration (memory, scheduler, disk).

With cloud providers, we obviously don't have access to this information, but we do need to perform a similar step, which consists in
studying the documentation and technical specifications of the instances (CPU, Burst, IOPS...).

## System activity review

I look at how the server or instance is behaving. Are storage, CPU and RAM saturated?

## Tuning Postgres

*Did you know that there are over 350 configuration parameters? The number increases (reasonably) with each new version. Some disappear, others appear.

Review of all Postgres configuration. The default configuration is very conservative, and needs to be reviewed.
Automatic configuration tools such as [PGTune](https://pgtune.leopard.in.ua/) are often inadequate and unsuitable.

*Tell yourself that if it were enough to apply simple rules, they would already be implemented in Postgres!

## Review of slow / power-hungry requests

Now that I have a better idea of the status of your instance, I can start inspecting slow and/or time-consuming queries.

It's usually at this step that you can expect the biggest performance gains.

Of course, I won't just point out the problematic queries. I'm going to suggest :

* Correct the query
* Creation of new indexes
* Review application behavior

## Schema review

Are any indexes missing? Are you using the right index type?

Are the types used correctly?

## Report

The review is a complete PDF report. I don't just apply corrections. I explain the corrections so that you can become more independent and avoid making the same mistakes.
