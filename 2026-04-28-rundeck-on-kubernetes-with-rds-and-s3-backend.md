---
title: Rundeck on Kubernetes with RDS and S3 Backend
slug: rundeck-on-kubernetes-with-rds-and-s3-backend
date_published: 2026-04-28T01:04:00.000Z
date_updated: 2026-04-29T01:07:34.000Z
excerpt: Rundeck deployed on EKS as a stateless job runner, using RDS for state and S3 for node inventory. Solves node loss from spot instances, uses a bastion for execution, and keeps setup minimal without introducing full CI/CD.
---

### Context and Goal

In one staging environment we needed a simple way to execute operational tasks across several machines without SSH-ing into each node manually. Typical tasks included restarting services, running maintenance scripts, or triggering operational workflows. The goal was not to build a full CI/CD platform, just to have a central place where engineers could execute tasks across infrastructure in a controlled way. For this, we deployed [Rundeck](https://www.rundeck.com/).

Rundeck fits this use case because the model is simple: define nodes, define jobs, and execute from a UI. There’s no pipeline abstraction or heavy integration layer. It works well when the problem is “run this across these machines” rather than “build a delivery system.”

---

### Environment and Constraints

The environment runs on Kubernetes (EKS), and an important constraint is that the cluster uses spot instances. That means pods can disappear at any time when underlying nodes are reclaimed. Because of that, we avoided storing anything important inside the cluster.

Rundeck is deployed as a stateless container (`rundeck/rundeck:5.16.0`), while all persistent data, projects, jobs, execution history, and configuration lives in PostgreSQL on RDS. Node inventory is handled separately.

---

### Database Setup

PostgreSQL is used as backend storage.

    create database rundeck;
    create user rundeckuser with password 'rundeckpassword';
    grant all privileges on database rundeck to rundeckuser;
    
    \c rundeck
    
    grant all privileges on schema public to rundeckuser;
    

For more details, please refer to [Rundeck Documentation](https://docs.rundeck.com/docs/administration/configuration/database/postgres.html#setup-rundeck-database)

---

#### Architecture

Rundeck runs inside Kubernetes and handles UI and execution logic. PostgreSQL stores all state including jobs, configuration, and execution history. Node definitions are external, and jobs typically execute against a reachable machine (often a bastion-style EC2 instance) that can access internal infrastructure. No persistent volume is attached to the Rundeck pod, which keeps it stateless but also exposed an issue early on.
![](__GHOST_URL__/content/images/2026/04/image-1.png)Rundeck - Simple Architecture
EKS in this architecture means the bastion host has access to the internal cluster and is used to run operational tasks. In our case, it’s used to scale down applications every night.
![](__GHOST_URL__/content/images/2026/04/image-2.png)Rundeck - Job History
---

#### Node Source Issue (and Fix)

Initially, node definitions were stored locally inside the pod as `resources.json`. This worked until pods restarted. Because the cluster runs on spot instances, node termination caused pod rescheduling, and the local filesystem disappeared. As a result, Rundeck suddenly showed no nodes in the UI and jobs couldn’t run.
![](__GHOST_URL__/content/images/2026/04/image-3.png)Rundeck - Nodes
This wasn’t a Rundeck issue; it was a bad assumption about persistence.

Instead of attaching a persistent volume just to keep a single JSON file, node definitions were moved to S3. The structure didn’t change:

    {
      "ec2-rundeck-client": {
        "nodename": "ec2-rundeck-client",
        "hostname": "xxx",
        "osVersion": "ubuntu-noble-24.04-arm64-server-20251001",
        "osFamily": "unix",
        "osArch": "aarch64",
        "ssh-key-storage-path": "keys/project/staging/ssh-privatekey",
        "description": "ec2-rundeck-client",
        "osName": "Linux",
        "tags": "ec2",
        "username": "ubuntu"
      }
    }
    

The file is uploaded to:

    s3://s3-poc-rundeck/resources.json
    

Then configured in Rundeck:

- Project Settings → Edit Nodes
- Add Source → AWS S3 Model Source
- Access Key / Secret Key → IAM credentials
- Region → ap-southeast-1
- Bucket → s3-poc-rundeck
- File → resources.json
- Writeable → Yes

Setting it as writeable allows Rundeck to update the node file directly from the UI. After moving to S3, the node disappearance issue stopped completely.

---

#### S3 Configuration Gotcha

One small issue: the `ap-southeast-3` AWS region wasn’t available in the Rundeck S3 dropdown. The workaround was to create the bucket in a nearby region (`ap-southeast-1`). Since the file is very small, cross-region latency is negligible. Not ideal, but acceptable.

IAM permissions used:

    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::s3-poc-rundeck",
        "arn:aws:s3:::s3-poc-rundeck/*"
      ]
    }
    

---

#### Notifications

Two mechanisms are currently used: SMTP for email alerts and a webhook integration handled by a small Node.js service. Rundeck sends execution events to the webhook, which can then process or forward them as needed. The webhook source code is available here: [https://github.com/anantafaturdev/rundeck-webhook](https://github.com/anantafaturdev/rundeck-webhook).
![](__GHOST_URL__/content/images/2026/04/image-4.png)Rundeck - Job Notifications
This implementation is simple but flexible enough to extend for additional notification channels or custom logic.
