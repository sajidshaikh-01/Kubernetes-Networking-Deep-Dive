# Cilium Network Policies – Types, Usage & Production Best Practices
---
## 1. What are Cilium Network Policies?

Cilium Network Policies control **who can talk to whom** inside a Kubernetes cluster.

Key difference from Kubernetes NetworkPolicy:

> **Kubernetes NetworkPolicy = L3/L4 (IP + Port)**
> **Cilium NetworkPolicy = L3–L7 (IP, Port, HTTP, gRPC, DNS, Kafka)**

Cilium policies are enforced using **eBPF in the Linux kernel**, not iptables.

---

## 2. Why Cilium Policies Are Different

Traditional Kubernetes NetworkPolicy:

* IP-based
* Port-based
* Hard to debug

Cilium NetworkPolicy:

* **Identity-based (labels, namespaces, service accounts)**
* Application-aware (HTTP paths, methods)
* Observable (drop reasons, flows)

👉 This is why Cilium is preferred in production clusters.

---

## 3. Types of Cilium Policies

Cilium supports **multiple policy CRDs** depending on scope and use case.

---

## 3.1 CiliumNetworkPolicy (CNP)

### Scope

* **Namespace-scoped**

### Used for

* Application-level policies
* Microservice communication control

### Example use cases

* Frontend → Backend only
* Backend → Database only

### Key characteristics

* Defined per namespace
* Uses labels for identity

---

## 3.2 CiliumClusterwideNetworkPolicy (CCNP)

### Scope

* **Cluster-wide**

### Used for

* Platform-level security
* Shared services
* Zero-trust defaults

### Example use cases

* Allow all namespaces to talk to CoreDNS
* Block all traffic except approved paths

### Key characteristics

* Applies across namespaces
* Managed by platform/SRE teams

---

## 3.3 Ingress Policies

Controls **incoming traffic** to Pods.

### Examples

* Allow traffic from frontend namespace
* Allow traffic from specific service account

Ingress rules define **who can talk TO a workload**.

---

## 3.4 Egress Policies

Controls **outgoing traffic** from Pods.

### Examples

* Allow backend to call database
* Allow application to access external APIs
* Block internet access by default

Egress control is **critical in production security**.

---

## 3.5 L7 (Application-Aware) Policies

Cilium can enforce policies at **Layer 7**.

### Supported protocols

* HTTP / HTTPS
* gRPC
* Kafka
* DNS

### Examples

* Allow only `GET /health`
* Block `POST /admin`
* Allow DNS queries only to approved domains

L7 policies are enforced using:

* eBPF + Envoy (transparent)

---

## 3.6 DNS Policies

Controls **which DNS names a Pod can resolve**.

### Why important

* Prevent data exfiltration
* Restrict external API access

### Example use cases

* Allow only `*.amazonaws.com`
* Block random internet domains

---

## 4. How Cilium Enforces Policies Internally

1. Pod sends packet
2. Packet intercepted by eBPF
3. Pod identity resolved (labels)
4. Policy lookup in eBPF maps
5. Decision made (allow / deny)
6. Event emitted to Hubble

👉 Enforcement happens **before packet leaves the node**.

---

## 5. How Cilium Policies Are Used in Production

### 5.1 Zero Trust Networking (Default Deny)

Production clusters usually:

* Deny all traffic by default
* Explicitly allow required paths

Pattern:

* CCNP → default deny
* CNP → app-specific allow rules

---

### 5.2 Platform vs Application Responsibility

| Team              | Policy Type |
| ----------------- | ----------- |
| Platform / SRE    | CCNP        |
| Application teams | CNP         |

This separation avoids security chaos.

---

### 5.3 Namespace-Based Isolation

* Each team gets its own namespace
* Policies prevent cross-namespace access

Used heavily in multi-tenant clusters.

---

### 5.4 East–West Traffic Control

Most production traffic is **east–west** (service-to-service).

Cilium policies are mainly used to:

* Control microservice communication
* Reduce blast radius

---

## 6. Cilium Policy Design Patterns (IMPORTANT)

### Pattern 1: Default Deny All

* CCNP denies all ingress/egress
* CNP explicitly allows traffic

Most secure pattern.

---

### Pattern 2: Allow DNS First

Always allow:

* kube-dns / CoreDNS

Otherwise applications will break.

---

### Pattern 3: Label-Based Identity

Use:

* app labels
* serviceAccount labels

Avoid IP-based rules.

---

### Pattern 4: Progressive Hardening

* Start with L3/L4
* Move to L7 gradually

Avoid breaking apps suddenly.

---

## 7. Production Best Practices (VERY IMPORTANT)

### ✅ Best Practices

1. Use **clusterwide default deny**
2. Separate platform and app policies
3. Prefer labels over namespaces alone
4. Enable **Hubble** for visibility
5. Version control all policies (GitOps)
6. Test policies in staging first
7. Document allowed flows

---

### ❌ Common Mistakes

❌ Writing very broad allow rules
❌ Skipping DNS policies
❌ Using IP-based rules
❌ Applying L7 policies everywhere
❌ No observability while enforcing

---

## 8. Cilium vs Kubernetes NetworkPolicy (Interview Table)

| Feature       | K8s NetworkPolicy | Cilium Policy |
| ------------- | ----------------- | ------------- |
| Enforcement   | iptables          | eBPF          |
| Identity      | IP-based          | Label-based   |
| L7 support    | ❌                 | ✅             |
| Observability | Limited           | Deep          |
| Performance   | Medium            | High          |

---

## 9. One-Line Interview Answer

> “Cilium network policies provide identity-based, L3–L7 security enforced using eBPF in the Linux kernel. In production, they are used to implement zero-trust networking, control east–west traffic, and provide deep observability through Hubble.”

