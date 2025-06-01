Title: How to Build a High-Performance Kubernetes Ingress for 1M+ RPS
Date: 2025-05-31
Category: Infrastructure
Tags: kubernetes, ingress, haproxy, scaling, aks, high-performance
Slug: kubernetes-ingress-scaling
Author: Jeff Palladino
Summary: Lessons from building a high-throughput HAProxy-based ingress layer for Kubernetes clusters handling millions of requests per second.

## Introduction

Handling millions of requests per second (RPS) through a Kubernetes cluster is not just a matter of adding replicas—it demands deliberate optimization of ingress design, connection handling, autoscaling, and network I/O. This post distills key strategies we used to scale an HAProxy-based ingress to consistently handle 1M+ RPS on Azure Kubernetes Service (AKS).

---

## Ingress Stack Overview

We used the following stack:
- **HAProxy** (with custom config map)
- **Azure Kubernetes Service (AKS)**
- **Horizontal Pod Autoscaler (HPA)**
- **Node pools tuned for low-latency**
- **Prometheus + Grafana** for observability

---

## HAProxy Configuration Essentials

In `ConfigMap`:
```yaml
maxconn-global: "100000"
maxconn-server: "10000"
nbthread: "4"
timeout-client: 50s
timeout-connect: 5s
timeout-queue: 5s
timeout-http-request: 10s
timeout-http-keep-alive: 1s
ssl-options: no-sslv3 no-tls-tickets no-tlsv10 no-tlsv11
ssl-ciphers: ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
```

These settings ensure fast connection handling, low latency, and strict SSL policies.

---

## Scaling Strategy

- Use **Dedicated Node Pools** for ingress controllers (separate from app workloads)
- Set **PodDisruptionBudgets** to avoid draining ingress under load
- Use **`topologySpreadConstraints`** or `podAntiAffinity` to prevent all ingress pods landing on one node

### HPA Tweaks

- Custom metric: `sum(rate(requests[1m]))`
- Stabilization window: 30s
- Cooldown: 60s

Ensure metrics server and Prometheus Adapter are tuned to avoid lag in metrics reporting.

---

## Connection and Network Limits

AKS nodes have system limits:
- **conntrack table**: Default is ~131072. You'll need to tune this with `sysctl` or use node images with extended limits
- **NIC throughput**: Scale with Standard_Dv4/Dv5 node series
- Watch out for `ConntrackFull` and `ReadOnlyFilesystem` errors on nodes under stress

---

## Observability

Key metrics to monitor:
- RPS per pod
- Latency P95/P99
- Dropped connections
- conntrack usage

Recommended tools:
- **Prometheus**: with `haproxy_exporter`
- **Grafana**: custom dashboards with alerts
- **Kubernetes Events**: monitor for pod eviction or failed scheduling

---

## Bonus: Simulate Load Without Overcommit

Use `wrk`, `vegeta`, or `k6` to simulate realistic traffic:
```bash
wrk -t12 -c1000 -d60s --latency https://your-ingress.example.com
```
This helps avoid triggering false autoscaler signals while still stressing the ingress layer.

---

## Conclusion

Building a high-throughput ingress isn’t just about more pods—it’s about smarter topology, system-level tuning, and proactive observability. With the right HAProxy configuration and node awareness, Kubernetes ingress can scale to serve millions of requests per second reliably.

Let me know if you'd like a Helm chart, Terraform config, or Azure-specific node tuning guide to go with this.
