---
title: "ssh: could not resolve hostname  - Kubernetes Jenkins Agent DNS Failure"
slug: ssh-could-not-resolve-hostname-kubernetes-jenkins-agent-dns-failure
date_published: 2026-03-27T14:56:53.000Z
date_updated: 2026-03-27T14:56:53.000Z
excerpt: Terraform init on Jenkins Kubernetes agents kept failing with "could not resolve hostname" when fetching terraform modules. Cluster DNS was flaky, so forcing pods to use 8.8.8.8 made it work.
---

We were setting up Jenkins from scratch for some reason. The setup is simple: Jenkins listens to push events from GitLab (`https://git.company.com`), then runs a pipeline to clone the repo and execute `terraform plan / apply / destroy`.

The flow is straightforward: clone repo → `terraform init` → `terraform plan/apply`.

The issue started after cloning. Repo cloning via SSH works fine, no problem at all. But when Terraform runs `init` to download modules (which are also hosted in the same GitLab), it fails with:

    ssh: could not resolve hostname: try again
    

This is confusing. Same GitLab domain, same SSH method, but behavior is different. Clone works, module download doesn’t.

We tried the usual things: restarting `tke-cni-agent`, `kube-proxy`, and `coredns`. No change. Still failing.

At this point we suspected DNS, but not sure how bad it was. So we spun up a debug daemonset across all 10 nodes using the same image, and ran `nslookup` to `git.company.com`. The result was random. Sometimes only 1–2 pods could resolve it, sometimes none, sometimes all worked. No consistent pattern.

So this is not Terraform, not Jenkins, not SSH. This is cluster DNS being unreliable under this condition. Terraform just exposed it because it does multiple fetches during `init`.

We didn’t try to fix DNS properly. That’s a different problem and would take longer. We just needed the pipeline to work.

So we forced the pod to use Google DNS instead of `coredns`:

    apiVersion: v1
    kind: Pod
    metadata:
      name: terraform pipeline
    spec:
    ######Fix DNS Issue######
      dnsPolicy: None
      dnsConfig:
        nameservers:
          - 8.8.8.8
        searches:
          - default.svc.cluster.local
    ######Fix DNS Issue######
    
      containers:
        - name: terraform
          image: hashicorp/terraform:1.14.3
          imagePullPolicy: Always
          command:
            - cat
          tty: true
          stdin: true
      securityContext:
        runAsUser: 0
        fsGroup: 0

By doing this, `/etc/resolv.conf` inside the container uses `8.8.8.8` instead of cluster DNS. After that, Terraform module downloads worked consistently.

This is obviously tech debt, but at least the pipeline works and we can deliver the MVP. Proper fix should be investigating `coredns` or network layer, but that’s for later (hopefully not me).
