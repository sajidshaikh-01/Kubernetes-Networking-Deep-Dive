# Kubernetes Services, Service Discovery & DNS – Production Guide

This README explains **how internal networking works in Kubernetes** from a **production + interview** perspective, covering:

* What is a Service
* What is Service Discovery
* What is DNS in Kubernetes
* Types of Services
* Endpoints & EndpointSlices
* How everything works together
* Troubleshooting internal networking issues

This is **vendor‑neutral** (works for EKS, AKS, GKE, Cilium, Calico, etc.).

---

## 1. What is a Kubernetes Service?

A **Service** is a Kubernetes abstraction that provides a **stable network identity** to access a group of Pods.

Why it is needed:

* Pod IPs are **ephemeral**
* Pods scale up/down
* Applications must not depend on Pod IPs

A Service provides:

* Stable **DNS name**
* Stable **virtual IP (ClusterIP)**
* **Load balancing** across Pods

Important:

> A Service itself does NOT forward packets. It defines rules that tell the data plane how to forward traffic.

---

## 2. What is Service Discovery?

**Service Discovery** is the mechanism that allows applications to **find and communicate with other services using names instead of IPs**.

In Kubernetes:

* Service Discovery is implemented using **Services + DNS**

Example:

```
frontend → orders-service → backend Pods
```

Even if Pods change, the Service remains stable.

---

## 3. What is DNS in Kubernetes?

**DNS (Domain Name System)** resolves:

```
service-name → IP address
```

In Kubernetes:

* DNS resolves **Service names → Service IPs**
* DNS does **NOT** load balance traffic
* DNS does **NOT** route packets

DNS only answers the question: *“What IP should I send traffic to?”*

---

## 4. Who Provides DNS in Kubernetes?

Kubernetes uses **CoreDNS**, which runs as Pods in the `kube-system` namespace.

CoreDNS:

* Watches the Kubernetes API
* Automatically creates DNS records for Services & Pods
* Responds to DNS queries from applications

If CoreDNS fails → **internal communication breaks**.

---

## 5. How Service Discovery Works (Step by Step)

Example:

* Service name: `orders`
* Namespace: `default`

Fully qualified DNS name:

```
orders.default.svc.cluster.local
```

Flow:

1. Application calls `http://orders`
2. DNS query goes to CoreDNS
3. CoreDNS returns Service ClusterIP
4. Application sends traffic to Service IP
5. Service rules forward traffic to a backend Pod

---

## 6. Types of Kubernetes Services

### 6.1 ClusterIP (Default)

* Exposes service **inside the cluster only**
* Most commonly used

Use cases:

* Microservice-to-microservice communication

---

### 6.2 NodePort

* Exposes service on **NodeIP:Port**
* Accessible externally

Use cases:

* Simple testing
* Rare in production directly

---

### 6.3 LoadBalancer

* Creates a **cloud load balancer**
* Traffic flow:
  External → LoadBalancer → Node → Pod

Use cases:

* Public-facing services
* Cloud environments (EKS, AKS, GKE)

---

### 6.4 Headless Service

* `clusterIP: None`
* No virtual IP
* DNS returns **Pod IPs directly**

Use cases:

* Databases
* StatefulSets
* Kafka, Elasticsearch

---

### 6.5 ExternalName

* Maps service name to an **external DNS name**

Use cases:

* External APIs
* Legacy systems

---

## 7. What are Endpoints?

**Endpoints** represent the **actual Pod IPs and ports** backing a Service.

For every Service:

* Kubernetes creates an Endpoints object
* It contains a list of Pod IPs

If no Endpoints exist:

* Service traffic goes nowhere

---

## 8. What are EndpointSlices?

**EndpointSlices** are the modern replacement for Endpoints.

Why they exist:

* Endpoints do not scale well
* Large Services caused performance issues

EndpointSlices:

* Split endpoints into smaller chunks
* Improve scalability
* Used automatically by Kubernetes

---

## 9. Service → Endpoint → Pod Flow

```
Pod
 ↓ (DNS)
Service Name
 ↓
Service ClusterIP
 ↓
Endpoint / EndpointSlice
 ↓
Backend Pod
```

Key point:

> If EndpointSlices are wrong, Services will not work.

---

## 10. Common Internal Networking Problems

### Problem 1: Service exists but traffic fails

Possible causes:

* No matching labels
* Endpoints empty
* Pods not Ready

Check:

```
kubectl get endpoints <svc>
kubectl get endpointslices
```

---

### Problem 2: DNS not resolving

Possible causes:

* CoreDNS down
* NetworkPolicy blocking DNS

Check:

```
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system coredns
```

---

### Problem 3: Service resolves but connection times out

Possible causes:

* NetworkPolicy / Cilium policy blocking traffic
* kube-proxy / eBPF issue

Check:

* Network policies
* CNI status

---

### Problem 4: Pod-to-Pod works but Pod-to-Service fails

Likely causes:

* kube-proxy broken
* CNI misconfiguration
* Service CIDR routing issue

---

## 11. Production Troubleshooting Checklist

1. Check Pod status (Running / Ready)
2. Check Service selectors
3. Check Endpoints / EndpointSlices
4. Verify DNS resolution
5. Check NetworkPolicies
6. Check CNI / kube-proxy / eBPF health

---

## 12. Interview Golden Rules

* DNS resolves names, does NOT load balance
* Services provide stable virtual IPs
* Endpoints map Services to Pods
* EndpointSlices improve scalability
* CNI moves packets, not Services

---


