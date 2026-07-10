---
title: "Experimental Setup: Cost-Optimized Site-to-Site VPN Using StrongSwan"
slug: experimental-setup-cost-optimized-site-to-site-vpn-using-strongswan
date_published: 2026-04-06T07:29:54.000Z
date_updated: 2026-04-06T07:29:54.000Z
excerpt: This experiment evaluates whether a low-cost, self-managed StrongSwan VPN on small ARM-based EC2 instances can provide sufficient performance and cost efficiency as an alternative to AWS Managed Site-to-Site VPN.
---

### **Why This Exists**

Managed Site-to-Site VPN provides convenience, but comes with a fixed hourly cost that may not be efficient for all workloads. This setup explores a self-managed VPN using StrongSwan on EC2, focusing on reducing cost while maintaining strong performance. The objective is to evaluate whether a small ARM-based instance can handle real traffic without becoming a bottleneck.

This exploration is currently implemented **within AWS only**, connecting **two separate VPCs**. While the configuration is structured to support multi-cloud scenarios such as AWS to GCP, that part has **not yet been implemented or validated**.

### **Scope & Limitations**

This setup connects two AWS VPCs using EC2 instances as VPN gateways. Latency testing was not performed. Some configurations such as restrictive security group rules and advanced routing were not deeply validated, since initial testing used permissive ingress. This should be treated as an exploratory baseline rather than a production-ready reference.

---

### **1. Architecture Overview**

This setup uses one EC2 instance in each VPC acting as a VPN gateway. These instances establish an IPsec tunnel using StrongSwan, allowing private CIDR communication across VPC boundaries.

    +---------------------+        IPsec VPN        +---------------------+
    |      AWS VPC A      | <---------------------> |      AWS VPC B      |
    |   10.0.0.0/16       |                         |   20.0.0.0/16       |
    |                     |                         |                     |
    |  EC2 (VPN Gateway)  |                         |  EC2 (VPN Gateway)  |
    |  10.0.0.242         |                         |  20.0.4.187         |
    |  3.109.85.41 (pub)  |                         |  13.233.242.92 (pub)|
    +---------------------+                         +---------------------+
    

Traffic between subnets is routed through these VPN instances via an encrypted tunnel.

### **2. Installation (Both Instances)**

StrongSwan must be installed on both instances. Additional plugins are included to support extended cryptographic features.

    sudo apt update
    sudo apt install strongswan strongswan-pki libcharon-extra-plugins -y
    

### **3. AWS VPC A Configuration**

This configuration defines the IPsec tunnel from VPC A to VPC B and is stored in `/etc/ipsec.conf`. It specifies local and remote endpoints, subnets, authentication method, and encryption algorithms. IKEv2 is used for key exchange, with multiple crypto proposals provided to improve compatibility across different environments.

    config setup
        charondebug="none"
        uniqueids=yes
        strictcrlpolicy=no
    
    conn %default
        ikelifetime=28800s
        lifetime=3600s
        rekeymargin=540s
        rekeyfuzz=100%
        keyingtries=3
        keyexchange=ikev2
        authby=psk
        mobike=no
    
    conn aws-a-to-b
        left=10.0.0.242
        leftid=3.109.85.41
        leftsubnet=10.0.0.0/16
        leftauth=psk
        leftfirewall=yes
    
        right=13.233.242.92
        rightid=13.233.242.92
        rightsubnet=20.0.0.0/16
        rightauth=psk
    
        auto=start
        type=tunnel
    
        dpdaction=restart
        dpddelay=30s
        dpdtimeout=120s
    
        ike=aes256-sha256-modp2048,aes256-sha256-modp3072,aes256-sha256-modp4096!
        esp=aes256-sha256,aes256-sha512,aes256gcm16!
    
        aggressive=no
        compress=no
        forceencaps=yes
    
        closeaction=restart
        replay_window=1024
    

### **4. AWS VPC B Configuration**

This configuration mirrors the setup from the opposite side and is also defined in `/etc/ipsec.conf`. The `left` and `right` roles are reversed, but all cryptographic parameters, lifetimes, and tunnel settings must remain consistent to ensure successful negotiation.

    config setup
        charondebug="none"
        uniqueids=yes
        strictcrlpolicy=no
    
    conn %default
        ikelifetime=28800s
        lifetime=3600s
        rekeymargin=540s
        rekeyfuzz=100%
        keyingtries=3
        keyexchange=ikev2
        authby=psk
        mobike=no
    
    conn aws-b-to-a
        left=20.0.4.187
        leftid=13.233.242.92
        leftsubnet=20.0.0.0/16
        leftauth=psk
        leftfirewall=yes
    
        right=3.109.85.41
        rightid=3.109.85.41
        rightsubnet=10.0.0.0/16
        rightauth=psk
    
        auto=start
        type=tunnel
    
        dpdaction=restart
        dpddelay=30s
        dpdtimeout=120s
    
        ike=aes256-sha256-modp2048,aes256-sha256-modp3072,aes256-sha256-modp4096!
        esp=aes256-sha256,aes256-sha512,aes256gcm16!
    
        aggressive=no
        compress=no
        forceencaps=yes
    
        closeaction=restart
        replay_window=1024
    

### **5. Pre-Shared Key Configuration**

StrongSwan separates authentication secrets from tunnel configuration. The Pre-Shared Key (PSK) is stored in `/etc/ipsec.secrets`, and must match on both sides. The identifiers used here must align with `leftid` and `rightid` in the main configuration.

Generate a strong key with `openssl rand -hex 32`, then apply on both instances:

    # /etc/ipsec.secrets
    3.109.85.41 13.233.242.92 : PSK "your-generated-key"
    

### **6. System Configuration**

IP forwarding must be enabled so the EC2 instance can route traffic between interfaces. IPv6 is disabled here to simplify the setup and avoid unintended routing behavior.

    sudo tee -a /etc/sysctl.conf << EOF
    net.ipv4.ip_forward=1
    net.ipv6.conf.all.disable_ipv6=1
    net.ipv6.conf.default.disable_ipv6=1
    EOF
    
    sudo sysctl -p
    

### **7. Routing**

Each VPC must know that traffic destined for the remote CIDR should be forwarded to the VPN instance. This is done through route tables.

**VPC A Route Table**

    Destination: 20.0.0.0/16
    Target: ENI of VPN EC2
    

**VPC B Route Table**

    Destination: 10.0.0.0/16
    Target: ENI of VPN EC2
    

### **8. Firewall Rules**

The VPN requires specific protocols and ports to function. These rules allow IKE negotiation, NAT traversal, and encrypted traffic flow between both endpoints.

    +----------+------+---------------------------+
    | Protocol | Port | Purpose                   |
    +----------+------+---------------------------+
    | UDP      | 500  | IKE (ISAKMP)              |
    | UDP      | 4500 | NAT Traversal             |
    | ESP      | 50   | Encrypted IPsec traffic   |
    | AH       | 51   | IPsec AH (optional)       |
    +----------+------+---------------------------+

These rules were not strictly tested, as the setup was validated using open inbound access (0.0.0.0/0) during the exploration phase. In a production environment, source IPs should be restricted to the peer VPN endpoint to reduce exposure. Protocol 50 (ESP) is required for carrying encrypted payloads, while Protocol 51 (AH) is optional and typically not used when ESP is enabled. It is also important to note that some cloud platforms require ESP and AH to be allowed as protocols rather than traditional port-based rules.

### **9. Start Service**

Once configuration is complete, StrongSwan must be restarted and enabled to start automatically.

    sudo systemctl restart strongswan-starter
    sudo systemctl enable strongswan-starter
    

### **10. Performance Benchmark**

To evaluate throughput, `iperf3` is used to generate traffic across the tunnel. Multiple parallel streams help simulate realistic load.

    #!/bin/bash
    
    TARGET_IP="10.0.0.242"
    DURATION=60
    LOGFILE="vpn_iperf_stress_$(date +%F_%H-%M-%S).log"
    
    while true; do
        iperf3 -c "$TARGET_IP" -t "$DURATION" -P 4 | tee -a "$LOGFILE"
    done

Across 10 runs, the total transferred data reached 146.0 GB, with an average throughput of approximately 1.91 Gbit/s using a `t4g.micro` instance. This indicates that even a small instance can achieve high throughput under burst conditions.

During this test, outbound traffic (egress) was billed for the full 146 GB transferred. The exact cost was not recorded, but this highlights an important consideration: while compute costs can be optimized, data transfer charges may become a significant portion of the total cost depending on traffic volume.

---

### **Cost Savings**

Managed VPN costs $36.50/month (calculated as $0.05/hour × 730 hours), while the self-managed setup costs $11.39/month (t4g.micro: $0.0106 × 730 = $7.74, Elastic IP: $0.005 × 730 = $3.65). This results in an estimated saving of $25.11/month, or approximately 69%. Even without including external cloud costs, this already demonstrates a strong cost advantage for simple VPN use cases. In practice, the cost can be optimized further, as the VPN service does not strictly require a dedicated EC2 instance and can be integrated into an existing instance such as a bastion host or utility server.

Another important consideration is the network performance characteristics of the instance type. The `t4g.micro` instance used in this setup provides a baseline bandwidth of approximately 87 Mbps, with the ability to burst up to around 2085 Mbps under favorable conditions. This explains how high throughput (around 1.91 Gbit/s) can be observed during testing, even on a small instance, as it leverages burst capability rather than sustained baseline performance. However, this performance is not guaranteed continuously, as it depends on available credits and workload patterns.

If a more consistent baseline throughput or higher sustained bandwidth is required, it is recommended to choose a different instance type with higher guaranteed network performance. AWS provides detailed specifications for each instance family, including baseline and maximum bandwidth, which can be referenced here: [https://docs.aws.amazon.com/ec2/latest/instancetypes/gp.html#gp_network](https://docs.aws.amazon.com/ec2/latest/instancetypes/gp.html#gp_network)

---

## **When This Approach Makes Sense**

This setup is suitable when cost optimization is important and there is willingness to manage infrastructure manually. It works well for low to medium traffic scenarios where performance is still required.

It is less suitable for environments requiring managed service guarantees, built-in high availability, or minimal operational overhead.
