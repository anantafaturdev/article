---
title: Connecting Azure and AWS with a Site-to-Site VPN
slug: connecting-azure-and-aws-with-a-site-to-site-vpn
date_published: 2026-04-13T04:38:15.000Z
date_updated: 2026-04-13T04:38:15.000Z
excerpt: A secure IPsec Site-to-Site VPN was established between AWS and Azure, enabling private encrypted connectivity between AWS VPC and Azure PostgreSQL subnet for cross-cloud access without any public IP exposure.
---

This setup establishes a secure IPsec Site-to-Site VPN between Azure and AWS, enabling private connectivity from AWS workloads to an Azure-hosted PostgreSQL database running in a restricted subnet.

A few months ago, the requirement was to connect the AWS environment with the Azure environment as part of a planned database migration from Azure PostgreSQL to AWS RDS. Since both environments were fully private with no public IP exposure, direct connectivity was not possible. The only viable approach was to establish a Site-to-Site VPN between the two clouds to allow controlled network-level access.

After the VPN connection was successfully established, the network integration itself worked as expected. However, during later stages of validation, the migration plan using AWS DMS was eventually cancelled due to changes in requirements. Even so, the hybrid connectivity setup remained useful and became a solid reference implementation for IPsec Site-to-Site VPN between AWS and Azure.

---

## 1. Azure Network Environment

The Azure resources are deployed under the resource group `service-kubernetes-production`. The main virtual network used is `service-production-aks-vnet`, which operates on the CIDR range `10.3.0.0/16`. Within this virtual network, a dedicated subnet named `pg-subnet-service-production` is configured with the CIDR `10.3.10.0/24`. This subnet is used exclusively for PostgreSQL workloads and is intentionally isolated as a private network segment without direct public access.

    +--------------------------------------------------------+
    | Azure: service-kubernetes-production                   |
    |                                                        |
    |  +------------------------------------------------+    |
    |  | Virtual Network: service-production-aks-vnet   |    |
    |  | CIDR: 10.3.0.0/16                              |    |
    |  |                                                |    |
    |  |   +--------------------------------------+     |    |
    |  |   | Subnet: pg-subnet-service-production |     |    |
    |  |   | CIDR: 10.3.10.0/24                   |     |    |
    |  |   |                                      |     |    |
    |  |   |  PostgreSQL (Private, no public IP)  |     |    |
    |  |   +--------------------------------------+     |    |
    |  |                                                |    |
    |  +------------------------------------------------+    |
    |                                                        |
    +--------------------------------------------------------+

Azure Architecture

---

## 2. Azure VPN Gateway Setup

To allow external connectivity from AWS, an Azure Virtual Network Gateway is provisioned. The gateway is named `vpn-azure-aws` and deployed in the Indonesia Central region using the VPN Gateway type with SKU `VpnGw2AZ` and Generation 2. It is associated with the virtual network `service-production-aks-vnet` and exposed through a public IP resource named `pip-vpn-azure-aws`. The public IP is dynamically assigned, and both active-active mode and BGP are disabled to keep the setup simple and static-route based. This gateway serves as the Azure-side endpoint for the IPsec tunnel.

---

## 3. AWS Network Environment

On the AWS side, the infrastructure is built around a VPC with CIDR `10.1.0.0/16`. To support VPN connectivity, a Virtual Private Gateway named `vpg-aws-azure` is attached to this VPC. In addition, a Customer Gateway is created in AWS using the public IP address of the Azure VPN Gateway. This establishes AWS’s reference point for the Azure-side endpoint.

    +------------------------------------------------------+
    | AWS Account / VPC                                    |
    | CIDR: 10.1.0.0/16                                    |
    |                                                      |
    |  +----------------------------------------------+    |
    |  | VPC: aws-production-vpc                      |    |
    |  |                                              |    |
    |  |   +--------------------+   +---------------+ |    |
    |  |   | Virtual Private    |   | Customer      | |    |
    |  |   | Gateway            |   | Gateway       | |    |
    |  |   | vpg-aws-azure      |   | cg-azure-vpn  | |    |
    |  |   +--------------------+   | (Azure VPN    | |    |
    |  |            |               |  Public IP)   | |    |
    |  |            |               +---------------+ |    |
    |  |            |                                 |    |
    |  |            +------ VPN Connection -----------+    |
    |  |                                              |    |
    |  +----------------------------------------------+    |
    |                                                      |
    +------------------------------------------------------+

AWS Architecture

---

## 4. AWS Site-to-Site VPN Configuration

A Site-to-Site VPN connection is then created in AWS with the name `vpn-aws-azure`. The connection uses the Virtual Private Gateway as the target and the previously defined Customer Gateway as the remote endpoint. Static routing is selected as the routing option, and the only advertised remote network is the Azure PostgreSQL subnet `10.3.10.0/24`, since that is the primary workload AWS needs to access. After creation, the VPN configuration file is downloaded using generic vendor settings with IKEv2. This file contains the tunnel outside IP addresses, pre-shared key, and IPsec configuration details required for Azure-side setup.

---

## 5. Azure Local Network Gateway Configuration

On Azure, a Local Network Gateway named `lng-azure-aws` is created to represent the AWS environment. It is deployed in the same resource group `service-kubernetes-production` and configured with the AWS VPC CIDR `10.1.0.0/16` as the address space. The gateway IP is set to the AWS VPN tunnel endpoint obtained from the downloaded AWS VPN configuration file. This step effectively tells Azure how to reach the AWS network through the VPN tunnel.

---

## 6. Azure VPN Connection Setup

Once both sides are defined, the VPN connection is established in Azure. A connection named `connection-azure-aws` is created between the Azure Virtual Network Gateway `vpn-azure-aws` and the Local Network Gateway `lng-azure-aws`. The connection uses Site-to-Site IPsec with IKEv2, and the pre-shared key is taken directly from the AWS VPN configuration file. This completes the Azure-side tunnel configuration and links both cloud environments.

---

## 7. AWS Route Table Configuration

To ensure traffic flows correctly between environments, the AWS route tables are updated. A static route is added for destination `10.3.10.0/24`, which represents the Azure PostgreSQL subnet, and the target is set to the Virtual Private Gateway `vpg-aws-azure`. This ensures that any traffic originating from AWS destined for Azure is correctly routed through the VPN tunnel.

---

## 8. Final Connectivity Outcome

After all configurations are completed and the tunnel becomes active, a secure hybrid connection is established between AWS and Azure. The AWS VPC `10.1.0.0/16` can now reach the Azure VNet `10.3.0.0/16`, specifically the PostgreSQL subnet `10.3.10.0/24`, over a private IPsec tunnel. This enables secure cross-cloud communication for use cases such as AWS DMS replication, hybrid application architectures, and data access patterns where AWS services need to interact with Azure-hosted databases.
