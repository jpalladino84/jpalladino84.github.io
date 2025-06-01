Title: How to Route Traffic Across Azure and Linode Using Equinix ExpressRoute
Date: 2025-05-31
Category: DevOps
Tags: azure, expressroute, linode, equinix, bgp, hybrid cloud
Slug: azure-linode-expressroute
Author: Jeff Palladino
Summary: A step-by-step engineering guide to configuring cross-cloud traffic between Azure and Linode using Equinix ExpressRoute and BGP.

## Introduction

Multi-cloud networking is complex, but necessary when you want to optimize cost, performance, or geographic redundancy. In this guide, I’ll walk through how I routed traffic from Azure to Linode through Equinix ExpressRoute, including the challenges, missteps, and lessons learned.

This setup enables low-latency, private, and reliable data transfer between Azure and Linode via Equinix’s fabric and BGP configuration.

---

## Architecture Overview

- **Azure**: Virtual Network (VNet) with ExpressRoute Gateway
- **Equinix**: Virtual Device and Fabric connection
- **Linode**: LKE (Linode Kubernetes Engine) with a private subnet
- **Protocol**: BGP over VLAN

### Key Goals:
- Avoid internet hops
- Enable deterministic routing
- Support redundancy

---

## Step 1: Provision ExpressRoute in Azure

1. Create a **Virtual Network Gateway** with `ExpressRoute` SKU
2. Link it to a subnet within your Azure VNet
3. Create an **ExpressRoute Circuit**
4. Choose **Equinix** as the provider and set the peering location

📌 Tip: Make sure you don’t enable Microsoft peering if you’re only routing to Linode. Use **Private peering**.

---

## Step 2: Create the Equinix Fabric Connection

1. Log into the [Equinix Fabric Portal](https://fabric.equinix.com/)
2. Create a **connection** between Azure and a virtual device (Cisco CSR or Palo Alto works well)
3. Assign VLAN tags to each side:
   - A-side: Azure (e.g., 10.10.1.1/30)
   - Z-side: Linode (e.g., 10.10.2.1/30)

💡 Watch for **overlapping subnets** between Azure and Linode! This caused initial route flaps.

---

## Step 3: Configure Linode for BGP

Linode doesn’t natively support BGP, so you’ll need to:

1. Deploy a router VM (e.g., VyOS or FRRouting)
2. Assign it a static IP on your private Linode subnet
3. Configure BGP neighbor with the Equinix Z-side
4. Advertise Linode CIDRs

---

## Step 4: Exchange Routes with Azure

In Azure:
- Use `Get-AzExpressRouteCircuit` to confirm peering is active
- Check learned routes via `Get-AzRouteTable`
- Ensure route tables are associated with subnets in your VNet

From Linode:
- Ping test Azure private IPs
- Run `traceroute` to confirm Equinix path is taken

---

## Common Pitfalls

- ❌ **Missing BGP ASN on one side** — causes peering rejection
- ❌ **Incorrect VLAN tags** — traffic drops silently
- ❌ **Unroutable return path** — remember to update **both** route tables
- ❌ **UDP/ICMP tests failing** — ExpressRoute doesn’t forward all protocol types by default

---

## Bonus: Test End-to-End Application Traffic

1. Deploy a sample app in Linode’s LKE
2. Use Azure Container App or App Gateway to hit the endpoint
3. Use `tcpdump` to confirm traffic path
4. Add latency monitoring via Prometheus or Grafana

---

## Conclusion

Routing traffic between Azure and Linode via Equinix ExpressRoute is absolutely possible—but requires surgical attention to BGP, subnets, and physical routing topology.

Once working, the benefits are huge: consistent performance, lower latency, and private, secure inter-cloud communication.

Let me know if you’d like a Terraform version of this configuration or a reusable BGP starter template!
