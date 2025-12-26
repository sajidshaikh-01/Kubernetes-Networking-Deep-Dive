# Kubernetes CNI (Container Network Interface)

## What is CNI?

**CNI (Container Network Interface)** is a **standard specification** that defines **how container runtimes should configure networking for containers and Pods**.

> Important:
>
> * CNI is **NOT a tool**
> * CNI is **a specification (contract)**
> * Networking is implemented by **CNI plugins**

Kubernetes **does not implement networking itself**. It relies on CNI-compliant plugins to:

* Assign IP addresses to Pods
* Connect Pods to the cluster network
* Enable Pod-to-Pod communication across nodes

---

## Why Do We Need CNI?

Kubernetes has **strict networking requirements**:

1. Every Pod must get its **own IP address**
2. Pods must communicate with **any other Pod directly** (no NAT)
3. Nodes must communicate with **all Pods**
4. IPs must be routable across nodes

Kubernetes core components **cannot fulfill this alone**.

That is why **CNI exists** — to allow **pluggable, vendor-independent networking**.

---

## What Are CNI Plugins?

**CNI plugins** are actual binaries that:

* Create network interfaces for Pods
* Assign IP addresses
* Configure routes and bridges
* Enforce network policies (if supported)

Examples (conceptual):

* Calico
* Cilium
* Flannel
* Weave

Each plugin implements the **same CNI specification**, but internally works differently.

---

## High-Level Architecture

```
+-------------------+
| Kubernetes API    |
+-------------------+
          |
          v
+-------------------+
| kubelet (Node)    |
+-------------------+
          |
          v
+-----------------------------+
| CNI Plugin (Binary)         |
|  - IP allocation (IPAM)    |
|  - Network setup           |
+-----------------------------+
          |
          v
+-----------------------------+
| Pod Network Namespace       |
|  eth0, IP, routes           |
+-----------------------------+
```

---

## How CNI Works – Step by Step (Pod Creation Flow)

### Step 1: Pod is Scheduled

* Kubernetes scheduler assigns the Pod to a Node

### Step 2: kubelet Creates Pod Sandbox

* kubelet asks the container runtime (containerd / CRI-O) to create Pod sandbox

### Step 3: kubelet Invokes CNI Plugin

* kubelet executes the CNI plugin binary
* Passes information via JSON:

  * Pod name
  * Namespace
  * Pod UID
  * Network namespace path

### Step 4: CNI Plugin Sets Up Networking

The plugin:

* Creates a **virtual interface (veth pair)**
* Attaches one end to the Pod network namespace
* Connects the other end to:

  * Linux bridge
  * Overlay network
  * eBPF dataplane (depends on plugin)

### Step 5: IP Address Assignment (IPAM)

* CNI plugin calls **IPAM**
* IPAM allocates an IP from Pod CIDR
* IP is assigned to Pod’s `eth0`

### Step 6: Routes Are Configured

* Routes are added so:

  * Pod can reach other Pods
  * Other Pods can reach this Pod

### Step 7: Pod is Ready

* kubelet reports Pod as **Running**

---

## CNI Plugin Internal Components

### 1. Main Plugin

Responsible for:

* Network interface creation
* Wiring Pod to network

### 2. IPAM Plugin

Responsible for:

* IP allocation
* IP release on Pod deletion

Example IPAM strategies:

* Host-local
* Kubernetes-based
* Cloud-provider based

---

## What Happens When Pod is Deleted?

1. kubelet calls CNI with `DEL` command
2. CNI plugin:

   * Removes network interfaces
   * Releases IP back to IPAM pool
3. Network resources are cleaned

---

## Where CNI Lives on Node

Common paths:

```
/etc/cni/net.d/      # CNI configuration files
/opt/cni/bin/        # CNI plugin binaries
```

---

## Relationship with Other Kubernetes Networking Concepts

| Component         | Responsibility                 |
| ----------------- | ------------------------------ |
| CNI               | Pod-to-Pod connectivity        |
| kube-proxy        | Service traffic routing        |
| CoreDNS           | Service discovery              |
| Ingress / Gateway | External traffic               |
| NetworkPolicy     | Traffic control (if supported) |

---

## Production Notes (Very Important)

* Without CNI → **Pods cannot communicate**
* Service Mesh (Istio) **still depends on CNI**
* NetworkPolicy only works if CNI supports it
* Performance & security heavily depend on CNI choice

---

## One-Line Interview Answer (Golden)

> "CNI is a specification that Kubernetes uses to delegate Pod networking to external plugins. The plugins implement IP allocation, routing, and connectivity so that Kubernetes networking requirements are satisfied."

---

## What You Should Do Practically Next

* Inspect CNI config:

  ```bash
  ls /etc/cni/net.d/
  ```
* Check Pod IPs:

  ```bash
  kubectl get pods -o wide
  ```
* Observe interfaces inside Pod:

  ```bash
  ip addr
  ```



