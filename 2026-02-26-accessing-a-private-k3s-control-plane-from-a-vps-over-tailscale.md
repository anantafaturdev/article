---
title: Accessing a Private K3s Control Plane from a VPS over Tailscale
slug: accessing-a-private-k3s-control-plane-from-a-vps-over-tailscale
date_published: 2026-02-26T07:09:05.000Z
date_updated: 2026-02-26T07:09:05.000Z
excerpt: A private K3s node on Proxmox VE is accessed from a public VPS via Tailscale. The VPS joins the mesh, and adding its Tailscale IP to K3s tls-san allows kubectl access without exposing port 6443 publicly.
---

I’m running a single-node K3s cluster inside my home lab on Proxmox VE. It’s intentionally boring: one control-plane node, no workers, no public IP. The VM sits on my LAN with `172.16.0.101`, and that’s it. No port forwarding, no exposure to the internet. The version is `v1.33.5+k3s1`, installed via the standard script.

Separately, I have a VPS running code-server. That VPS has a public IP, but I don’t want to expose Kubernetes API port 6443 publicly just so I can run `kubectl`. The requirement is simple: from the VPS, I want to run `kubectl get nodes` against the home K3s control plane.

I was already using Tailscale to access my homelab remotely from my laptop. So instead of inventing something new, I just extended the same mesh to include the VPS. On both machines, I installed Tailscale:

    curl -fsSL https://tailscale.com/install.sh | sh
    sudo tailscale up
    

Both nodes authenticated into the same Tailscale account. On the K3s node, I checked the assigned IP:

    tailscale ip -4
    

It returned:

    100.76.241.16
    

From the VPS, I tested connectivity:

    ping 100.76.241.16
    

It worked immediately. I didn’t touch any firewall rules on either side. As long as both machines are inside the same Tailscale network, the mesh handles the routing. That part was uneventful.

Next step was kubeconfig. On the K3s node, the config file lives at:

    /etc/rancher/k3s/k3s.yaml
    

I copied it manually to the VPS. Inside that file, the `server:` field originally pointed to localhost:

    server: https://127.0.0.1:6443
    

I changed it to the Tailscale IP:

    server: https://100.76.241.16:6443
    

At this point I expected it to just work. Network connectivity was there, port 6443 was reachable over the mesh, and nothing was blocked.

Then I ran:

    kubectl get nodes
    

And got:

    x509: certificate is valid for 127.0.0.1, 172.16.0.101, not 100.76.241.16
    

This is where it gets interesting. The failure wasn’t network-related. It was TLS. K3s had generated its API server certificate with Subject Alternative Names that included `127.0.0.1` and the LAN IP `172.16.0.101`, but not the Tailscale IP. So when I connected using `100.76.241.16`, the certificate validation failed exactly as it should.

The fix is straightforward but easy to forget. On the K3s node, I edited:

    /etc/rancher/k3s/config.yaml
    

and added:

    tls-san:
      - 127.0.0.1
      - 172.16.0.101
      - 100.76.241.16
    

Then restarted K3s:

    systemctl restart k3s
    

K3s regenerated the serving certificate with the updated SAN entries. No manual certificate work. No reinstallation. It handled it cleanly.

Back on the VPS, I ran again:

    kubectl get nodes
    

This time the output was:

    NAME         STATUS   ROLES                       AGE    VERSION
    k3s-master   Ready    control-plane,etcd,master   141d   v1.33.5+k3s1
    

That’s it. No public IP. No port forwarding. No firewall rules changed. The VPS talks to the control plane entirely over the Tailscale mesh, and TLS is properly validated.
