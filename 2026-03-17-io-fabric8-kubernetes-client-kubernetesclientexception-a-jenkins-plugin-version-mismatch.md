---
title: "io.fabric8.kubernetes.client.KubernetesClientException: A Jenkins Plugin Version Mismatch"
slug: io-fabric8-kubernetes-client-kubernetesclientexception-a-jenkins-plugin-version-mismatch
date_published: 2026-03-17T16:39:18.000Z
date_updated: 2026-03-17T16:39:18.000Z
excerpt: Connection test from old Jenkins (2.492.2) to a new Kubernetes cluster failed with io.fabric8.kubernetes.client.KubernetesClientException due to Unrecognized field "emulationMajor"; upgrading the Kubernetes plugin fixed it.
---

### Issue Overview

We hit an issue while trying to add a new Kubernetes cluster into an existing Jenkins setup. During “Test Connection” in Jenkins cloud config, it failed with: `io.fabric8.kubernetes.client.KubernetesClientException`

At first glance, this looked like a typical infra issue. Firewall, RBAC, credentials, something along those lines. But the behavior didn’t match. The same cluster and credentials worked perfectly when tested from another Jenkins environment. That was the first hint this wasn’t about access.

### Jenkins & Kubernetes Setup

The setup itself matters here. We have two Jenkins environments:

- Old Jenkins → `2.492.2`, Java `17`, kubernetes plugin `4306.vc91e951ea_eb_d`, kubernetes-client-api `6.10.0-251.v556f5f100500`
- New Jenkins → `2.546`, Java `21`, kubernetes plugin `4416.v2ea_b_5372da_a_e`, kubernetes-client-api `7.3.1-256.v788a_0b_787114`

And two kubernetes clusters:

- Old cluster → `v1.32.12`
- New cluster → `v1.34.1`

Behavior was consistent:

- Old cluster → Old Jenkins → works
- New cluster → New Jenkins → works
- New cluster → Old Jenkins → fails

Same endpoint, same credentials. So we ruled out network, firewall, and RBAC pretty early, even though we still double-checked out of habit. From the Jenkins VM, a simple: `curl https://<k8s-api>/version` returned a valid response. That confirmed the API was reachable and TLS was fine. The failure was happening inside Jenkins itself.

### Investigation & Clues

The real clue came from the logs: `Unrecognized field "emulationMajor" (class io.fabric8.kubernetes.client.VersionInfo)`

That’s not connectivity. That’s parsing failure. So we checked the `/version` endpoint manually.

Old cluster response:

    {
      "major": "1",
      "minor": "32",
      "gitVersion": "v1.32.12"
    }
    

New cluster response:

    {
      "major": "1",
      "minor": "34",
      "emulationMajor": "1",
      "emulationMinor": "34",
      "minCompatibilityMajor": "1",
      "minCompatibilityMinor": "33",
      "gitVersion": "v1.34.1"
    }
    

The new cluster introduces additional fields like `emulationMajor` and `minCompatibilityMajor`. The older Fabric8 client used by `kubernetes-client-api 6.10.0` doesn’t recognize these fields and fails during deserialization. It doesn’t ignore unknown fields, it just throws an exception. That’s why the connection test fails even though everything else is fine.

This also explains why the new Jenkins works. It uses `kubernetes-client-api 7.3.1`, which already supports the updated API response. Same cluster, same credentials, different client version, different result.

### Solution

The fix was straightforward once the root cause was clear. We upgraded the Kubernetes plugin in the old Jenkins. Through incremental testing, we found that versions below `4355.v37e9e7c240e6` still fail, and starting from that version the issue is resolved. We didn’t jump straight to latest, just moved step by step until it worked.

The upgrade wasn’t isolated. Jenkins pulled a chain of dependencies along with it:

- **Kubernetes Client API**
- Old Version: `6.10.0-251.v556f5f100500`
- Upgraded Version: `7.3.1-256.v788a_0b_787114`

- **Kubernetes**
- Old Version: `4306.vc91e951ea_eb_d`
- Upgraded Version: `4355.v37e9e7c240e6`

- **Docker Commons**
- Old Version: `451.vd12c371eeeb_3`
- Upgraded Version: `457.v0f62a_94f11a_3`

- **Kubernetes Credentials**
- Old Version: `192.v4d5b_1c429d17`
- Upgraded Version: `203.v85b_9836a_f44b_`

- **Folders Plugin (cloudbees-folder)**
- Old Version: `6.1012.v79a_86a_1ea_c1f`
- Upgraded Version: `6.1026.ve06dfa_cf31c3`

- **Metrics**
- Old Version: `4.2.30-471.v55fa_495f2b_f5`
- Upgraded Version: `4.2.32-476.v5042e1c1edd7`

We didn’t selectively upgrade plugins. This followed Jenkins dependency resolution. After the upgrade, both clusters connected successfully:

- **Old cluster:** Status: Connected, Version: v1.32.12
- **New cluster:** Status: Connected, Version: v1.34.1

Looking back, the main mistake was assuming this was an infrastructure issue. We spent time checking things that were already implicitly proven working. The error message itself was the strongest signal, but easy to overlook because it doesn’t look like a typical Kubernetes failure.

> One important note: all of this was done in staging. We haven’t rolled the plugin upgrade into production yet. Upgrading Jenkins plugins can have side effects on pipelines and other integrations.
