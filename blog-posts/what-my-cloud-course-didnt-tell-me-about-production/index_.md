---
title: "What My Cloud Course Didn't Tell Me About Production"
description: "Bridging the AWS theory from my UTS cloud unit and a real deployment — through the lens of someone who has to keep things running."
pubDate: 2026-06-12
tags: ["AWS", "DevOps", "Cloud", "Production Support"]
draft: false
---



In a lab exercise, an EC2 instance is a thing you create, SSH into, and terminate before the marks are tallied. In production, an EC2 instance is a thing that someone is depending on at 2am while you read logs trying to work out why it stopped talking to the database.

My UTS cloud unit gave me the vocabulary — IaaS vs PaaS vs SaaS, the shared-responsibility model, the NIST definition of what "cloud" even means. What it could not give me was the feeling of being the person responsible when the diagram stops being a diagram and starts being an incident. I picked most of that up the slow way: running my own services, and breaking them.

This post is the bridge between the two. I'll take a fairly ordinary AWS application, walk through it the way the coursework describes it, and then walk through it again the way you actually experience it when something is wrong.

## The architecture (the version from the slides)

A typical small AWS application looks like this:

- **Compute** — EC2 instances, or containers running on them, serving the application.
- **Database** — a managed relational database on RDS, so you are not patching Postgres yourself at midnight.
- **Storage** — S3 for files, backups, and anything you do not want sitting on the instance's disk.
- **Messaging** — SQS or Amazon MQ to decouple the parts of the system that should not have to wait for each other.
- **Observability** — CloudWatch collecting logs and metrics from all of it.

> *[architecture diagram goes here — request, EC2/containers, RDS, S3, SQS, CloudWatch]*

On the slides this is clean. Each box is a managed service, each arrow is a well-documented API. The shared-responsibility model says AWS keeps the boxes alive and I keep what runs inside them healthy. All true. None of it tells you where the system actually fails.

## The architecture (the version you live with)

The interesting part of any system is not how it works when everything is fine — it's the small number of ways it reliably goes wrong. Here are three I've actually run into, in one form or another.

### 1. "The app can't reach the database"

When I deployed a WireGuard VPN on an EC2 instance, the first thing that bit me had nothing to do with WireGuard. It was the **security group**. The service was configured perfectly and refused all connections, because the inbound rule for the UDP port simply wasn't there. The logs on the instance looked healthy — the failure was one layer out, in the network configuration, where there was nothing to log.

The same shape shows up with RDS. "Connection refused" or, worse, a connection that just hangs, is almost never the database being down. It's a security group that doesn't allow the app's subnet, credentials that rotated, or a connection pool that's exhausted because something upstream isn't releasing connections. The skill isn't knowing the fix — it's knowing the *order to check things in* so you're not guessing.

### 2. "The queue is backing up"

A message queue is wonderful until the consumer dies quietly and the queue length climbs while nothing visibly breaks. Then a "poison message" — one malformed payload the consumer can't process — gets retried forever, blocking everything behind it. Nobody filed a bug, because from the outside the system still accepts requests. It just stops *finishing* them.

This is the kind of failure that taught me to care about queue depth as a first-class metric, not an afterthought.

### 3. "A container is crash-looping"

In my homelab I run fifteen-odd services in Docker behind Traefik, deployed GitOps-style through CI. The most common failure is the least dramatic: a container that starts, fails a health check, and restarts, over and over, fast enough that by the time you look it's "running" again. The signal is not in the current state. It's in the *restart count* and the last few log lines before each death.

The pattern across all three: **the broken thing and the visible symptom are usually in different places.** Production support is mostly the discipline of closing that gap quickly.

## Seeing the problem before the client tells you

You can't fix what you can't see, and you can't triage what you didn't instrument. This is where the cloud course and the homelab finally met for me.

- **CloudWatch** is the native answer on AWS — centralised logs, metrics, and alarms. The lesson I learned the hard way: an alarm on the *symptom* (error rate, queue depth, connection count) is worth more than ten dashboards you only look at after someone complains.
- In my own infrastructure I lean on **Prometheus and Grafana** for the same job — scraping metrics, drawing the trends, and alerting when a number crosses a line it shouldn't.

The tool matters less than the habit. Whether it's CloudWatch or Grafana, the goal is the same: shrink the time between "something is wrong" and "I know which box went red."

## The part that's actually the job

Once you can see it, resolution is a judgement call I think of in three buckets:

1. **Config / environment** — security group, credentials, a wrong environment variable. Fix it, document what it was, move on.
2. **Operational** — restart the stuck consumer, drain the poison message, scale out. Treat the cause, not just the symptom, or it comes back.
3. **A real defect** — the code is wrong. This is where you stop fixing and start *writing it up*: reproduction steps, the relevant log lines, what you ruled out, and a clear hand-off so a developer can pick it up without re-doing your investigation.

That third bucket is the one I underestimated. A good investigation note — "here's what happens, here's why, here's the evidence" — is worth more to a team than a fast but undocumented fix. It's the difference between resolving an issue and resolving it *once*.

## What I'm learning next

Two things I'm deliberately working on:

- **Infrastructure as Code** — moving from clicking around the console to defining infrastructure in Terraform, so the environment is reproducible and the "what changed?" question has an answer in version control.
- **Kubernetes** — my homelab is Docker and Traefik today; the natural next step is understanding how this scales when one host stops being enough.

The through-line, though, hasn't changed since the first time a security group quietly ate my traffic: a cloud architecture is only as good as your ability to operate it when it misbehaves. The course taught me the boxes and the arrows. Production taught me to assume one of them will eventually lie to me — and to know where to look when it does.