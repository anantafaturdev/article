---
title: Setting up Jenkins on Kubernetes via Helm (and Fixing Restart Issues)
slug: setting-up-jenkins-on-kubernetes-via-helm-and-fixing-restart-issues
date_published: 2026-02-25T04:31:43.000Z
date_updated: 2026-02-25T08:06:46.000Z
excerpt: Installing Jenkins on Kubernetes via Helm worked smoothly at first, but after a node restart, the pod became unhealthy during plugin initialization. The init container attempted to re-copy plugins into an already populated persistent volume, causing overwrite prompts and blocking startup.
---

This was a small POC, not production. The goal was simple: regularly clean old GitLab artifacts without touching the existing GitLab pipeline. I initially considered adding a new cleanup stage inside the existing pipeline, but as the most junior in the room, modifying shared CI configuration felt risky. If I broke something, it would affect everyone. We already had Jenkins in our infra, so I proposed using a scheduled Jenkins pipeline to handle artifact cleanup instead.

### Installing Jenkins via Helm

Jenkins provides a maintained Helm chart at `https://charts.jenkins.io`, I added the repo and installed it:

    helm repo add jenkins https://charts.jenkins.io
    helm repo update

In theory, we can just run: `helm upgrade --install jenkins jenkins/jenkins`. And yes, that works initially. Jenkins comes up, you log in, everything looks fine. But after a cluster or node restart, the Jenkins pod did not behave cleanly.

### The Problem After Restart

After restarting the k3s node, the Jenkins pod came back in an unhealthy state, looking at the init container logs: `kubectl logs jenkins-0 -n jenkins -c init`.

    Setting checksum for: configuration-as-code to xA895ZNcY2HZoS9kNLE9KX+ew6VaI3tPoljFbNGTJQA=
    Setting checksum for: configuration-as-code to xA895ZNcY2HZoS9kNLE9KX+ew6VaI3tPoljFbNGTJQA=
    ...
    Done copy plugins to shared volume
    cp: overwrite '/var/jenkins_plugins/antisamy-markup-formatter.jpi'?
    cp: overwrite '/var/jenkins_plugins/apache-httpcomponents-client-4-api.jpi'?
    cp: overwrite '/var/jenkins_plugins/asm-api.jpi'?
    ...
    cp: overwrite '/var/jenkins_plugins/workflow-support.jpi'?
    

The init container was trying to copy plugins again into `/var/jenkins_plugins`. Since we were using persistence (default chart behavior with PVC), the plugin directory already existed from the previous run. On restart, the init container re-ran and attempted to overwrite existing plugin files.

This looked like re-initialization behavior rather than PVC corruption. Storage was local-path on a single node, so there was no cross-node scheduling issue. The PVC was reused correctly. The friction was coming from the initialization logic itself.

### Inspecting and Modifying values.yaml

Instead of guessing Helm chart behavior, I dumped the chart values:

    helm show values jenkins/jenkins >> jenkins-values.yaml
    

After reviewing the configuration, I found:

    initializeOnce: false
    

The default behavior allows initialization logic to run more than once. For a persistent controller, that is not what I wanted. So I changed it:

    sed -i 's/initializeOnce: false/initializeOnce: true/' jenkins-values.yaml
    

With `initializeOnce: true`, the initialization phase runs only on the first startup. On subsequent pod restarts, it skips re-initialization. After this change, restarting the node no longer triggered repeated plugin copy attempts.

Then I also set the external URL to avoid reverse proxy warnings:

    sed -i 's|jenkinsUrl:.*|jenkinsUrl: https://jenkins.anantafatur.dev|' jenkins-values.yaml
    

Finally, I applied the configuration:

    helm upgrade --install jenkins jenkins/jenkins \
      --namespace jenkins \
      --create-namespace \
      -f jenkins-values.yaml
    

After this, restarts were clean. No repeated plugin copy prompts, no initialization noise.

### Retrieving the Admin Password

To retrieve the initial admin password:

    kubectl get secret jenkins -n jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode && echo
    

Example output:

    Bfh8gMi4AERZDRiQoedsef
    

For now, this approach gave me a safe playground to experiment without touching shared CI configuration.
