---
title: Jenkins on Docker (VM) with Kubernetes Agents
slug: jenkins-on-docker-vm-with-kubernetes-agents
date_published: 2026-04-13T10:10:23.000Z
date_updated: 2026-04-13T10:10:59.000Z
excerpt: In this write-up, I’ll walk through how I set up Jenkins running on Docker on a VM and integrated it with Kubernetes, until it’s able to provision pods directly into the cluster.
---

The main constraint is that Jenkins must run on a VM. Moving it to a fully Kubernetes-managed setup was not an option, so the compromise was to keep the controller on VM and offload workloads to Kubernetes agents.

The Kubernetes cluster in this setup is managed (TKE), which means there is no direct access to nodes. Everything is done through the Kubernetes API, so authentication and RBAC become the critical parts of the integration.

---

# Jenkins Server Setup (VM-Based)

I started from a clean setup using the official `jenkins/jenkins:2.546` image. I didn’t build a custom image because I wanted to keep the setup minimal and avoid introducing unnecessary variables early on.

Instead of running containers manually, I used Docker Compose to make the setup easier to manage.

    version: "3.8"
    
    services:
      jenkins:
        image: jenkins/jenkins:2.546
        container_name: jenkins
        restart: always
        ports:
          - "8080:8080"
          - "50000:50000"
        environment:
          - JAVA_OPTS=-Xms1024m -Xmx1024m
          - TZ=Asia/Jakarta
        volumes:
          - /root/data/jenkins/home:/var/jenkins_home
          - /root/data/jenkins/logs:/var/log/jenkins
          - /var/run/docker.sock:/var/run/docker.sock
          - /usr/bin/docker:/usr/bin/docker
          - /etc/localtime:/etc/localtime:ro
          - /etc/timezone:/etc/timezone:ro
    

The important detail here is persistence. Jenkins stores everything in `/var/jenkins_home`, so that directory is mapped to the host. Without this, all configuration would be lost on restart.

Mounting the Docker socket allows Jenkins jobs to run Docker commands directly. This is convenient but gives the container high privileges on the host, so it’s a trade-off I accepted for this setup.

Timezone mismatch was another small but annoying issue. By default, Jenkins runs in UTC, which makes logs confusing. Setting `TZ` and mounting system time files fixes that.

After the container is up, ports 8080 and 50000 need to be accessible. If port 50000 is blocked, Kubernetes agents will fail to connect later, and the error is not immediately obvious.

---

## Backup Strategy

Since Jenkins state is stored on disk, I added snapshot-based backups using Tencent Cloud CBS.

    resource "tencentcloud_cbs_snapshot_policy" "jenkins_server" {
      snapshot_policy_name = "snapshot-policy-jenkins-server"
      repeat_weekdays      = [0,1,2,3,4,5,6]
      repeat_hours         = [2]
      retention_days       = 3
    }
    
    resource "tencentcloud_cbs_snapshot_policy_attachment" "jenkins_server" {
      storage_id         = "disk-xxxxxxxx"
      snapshot_policy_id = tencentcloud_cbs_snapshot_policy.jenkins_server.id
    }
    

This gives a simple daily backup with short retention, enough for recovery from most operational mistakes.

---

# Kubernetes as Jenkins Agent

Before configuring Jenkins, I prepared the Kubernetes side first.

I verified cluster connectivity with `kubectl get nodes`. All nodes must be in `Ready` state. If not, agents will fail to schedule later.

---

## Namespace, Service Accounts, and RBAC

I created a dedicated namespace to isolate Jenkins workloads, along with two service accounts: one for Jenkins and one for Terraform jobs.

    apiVersion: v1
    kind: Namespace
    metadata:
      name: jenkins-agent
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: jenkins
      namespace: jenkins-agent
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: terraform
      namespace: jenkins-agent
    

Then I defined RBAC permissions so Jenkins can create and manage pods:

    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: jenkins-agent-role
      namespace: jenkins-agent
    rules:
    - apiGroups: [""]
      resources: ["pods", "pods/log", "pods/exec", "secrets", "services"]
      verbs: ["get", "list", "watch", "create", "delete"]
    - apiGroups: ["batch"]
      resources: ["jobs"]
      verbs: ["get", "list", "watch", "create", "delete"]
    - apiGroups: ["apps"]
      resources: ["deployments"]
      verbs: ["get", "list", "watch", "create", "delete"]
    

And bound it to the Jenkins service account:

    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: jenkins-agent-rolebinding
      namespace: jenkins-agent
    subjects:
    - kind: ServiceAccount
      name: jenkins
    roleRef:
      kind: Role
      name: jenkins-agent-role
      apiGroup: rbac.authorization.k8s.io
    

Terraform gets broader permissions because it modifies resources.

After that, I applied everything: `kubectl apply -f manifest.yaml`

---

## Private Registry Secret

To allow pulling images from a private registry:

    kubectl create secret docker-registry dockerhub \
      --docker-server=YOUR_REGISTRY \
      --docker-username=USERNAME \
      --docker-password='PASSWORD' \
      -n jenkins-agent
    

Without this, pods will fail with `ImagePullBackOff`.

---

# Adding Kubernetes Credentials into Jenkins

This is the step that ties everything together.

Jenkins needs credentials to authenticate to the Kubernetes API. Instead of using kubeconfig, I used a service account token.

First, I retrieved the token:

    kubectl get secret jenkins-sa-token \
      -n jenkins-agent \
      -o jsonpath='{.data.token}' | base64 -d
    

Then I retrieved the CA certificate:

    kubectl get secret terraform-sa-token \
      -n jenkins-agent \
      -o jsonpath='{.data.ca\.crt}' | base64 -d
    

With these two values, I went into Jenkins UI.

From the dashboard, I navigated to:

    Manage Jenkins → Manage Credentials
    

I added a new credential with the following configuration:

- Kind: **Secret Text**
- Secret: (paste the service account token)
- ID: `jenkins-sa-token`
- Description: Kubernetes service account token

One thing to note is that Jenkins doesn’t validate this at creation time. If the token is wrong, you only find out later when the Kubernetes cloud fails to connect.

---

# Configuring Kubernetes Cloud in Jenkins

After adding the credentials, I configured the Kubernetes cloud.

Navigate to: `Manage Jenkins → Manage Nodes and Clouds → Configure Clouds`

Then add a new Kubernetes cloud configuration.

The key fields are:

- Kubernetes URL → cluster endpoint (usually from load balancer)
- Credentials → the `jenkins-sa-token` created earlier
- Kubernetes Namespace → `jenkins-agent`
- Kubernetes server certificate key → paste the CA certificate

After saving, I used the “Test Connection” button.

If everything is correct, Jenkins reports a successful connection to the cluster. In my case, it showed something like:

    Connected to Kubernetes v1.34.x
    

If it fails, the most common issues I hit were:

- Token mismatch or expired token
- Missing RBAC permissions
- Wrong API endpoint
- Security group blocking access

---

# Result

At this point, Jenkins is fully connected to Kubernetes. Each job that uses the Kubernetes plugin will dynamically provision a pod in the `jenkins-agent` namespace. Once the job finishes, the pod is deleted automatically. This keeps the system clean and avoids resource buildup.
