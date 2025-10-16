# Cloud Computing Networking Concepts


## Table of Contents

1. [TUN Interface](#1-tun-interface)  
2. [TAP Interface](#2-tap-interface)  
3. [Open vSwitch (OVS)](#3-open-vswitch-ovs)  
4. [VXLAN](#4-vxlan)  
5. [Flannel (CNI)](#5-flannel-cni)  
6. [Cilium](#6-cilium)  
7. [Packet Copy](#7-packet-copy)  
8. [Zero Copy](#8-zero-copy)  
9. [User Space to Kernel Space Packet Route](#9-user-space-to-kernel-space-packet-route)  
10. [Libvirt](#10-libvirt)  
11. [Nodes and Pods in Kubernetes (K8s)](#11-nodes-and-pods-in-kubernetes-k8s)

---

## 1. TUN Interface
**TUN (Network TUNnel)** is a virtual network interface that operates at **Layer 3 (Network Layer)**.  
It handles **IP packets** and passes them to a user-space program instead of the physical network.  
This allows custom processing like encryption, encapsulation, or routing.

**Use Case:** VPNs such as OpenVPN or WireGuard use TUN to securely tunnel IP packets.  
**Example Device:** `/dev/net/tun`

---

## 2. TAP Interface
**TAP (Network TAP)** works at **Layer 2 (Data Link Layer)** and handles **Ethernet frames**.  
It acts as a virtual Ethernet device that can connect virtual machines or containers to software bridges.

**Use Case:** Virtualization software like QEMU uses TAP interfaces to connect VMs to the same network bridge.  
**Difference:**  
- TUN → IP packets (L3)  
- TAP → Ethernet frames (L2)

---

## 3. Open vSwitch (OVS)
**Open vSwitch (OVS)** is a **software-based virtual switch** that connects virtual network interfaces (like TAP/TUN or veth pairs).  
It enables programmable, SDN-controlled networking within virtualized hosts.

**Key Features:**  
- VLAN & VXLAN support  
- Flow-based forwarding  
- Integration with SDN controllers  
- Kernel datapath acceleration  

**Use Case:**  
Used in **OpenStack**, **Kubernetes**, and **NFV** environments for dynamic virtual networking.

---

## 4. VXLAN (Virtual Extensible LAN)
**VXLAN** extends Layer 2 networks over Layer 3 using UDP encapsulation.  
It allows isolated networks to span multiple physical hosts by wrapping Ethernet frames inside IP/UDP packets.

**VXLAN Header:**  
Includes a **24-bit VXLAN Network Identifier (VNI)** → supports up to 16 million logical networks.

**Use Case:**  
Large-scale data centers and Kubernetes CNIs (like Flannel or Calico) use VXLAN to connect pods or VMs across different hosts.

---

## 5. Flannel (CNI)
**Flannel** is a lightweight **Container Network Interface (CNI)** plugin for Kubernetes.  
It provides an **overlay network** that connects pods across multiple nodes.

**Working Principle:**
- Each node gets a unique **subnet**.  
- Flannel encapsulates pod traffic (usually using VXLAN).  
- Encapsulated packets are routed between nodes.

**Use Case:**  
Used for simple and easy Kubernetes networking setups.

---

## 6. Cilium
**Cilium** is an advanced **CNI** that uses **eBPF (Extended Berkeley Packet Filter)** to achieve high-performance networking and deep security.  

**Advantages:**
- No reliance on `iptables` (uses eBPF filters).  
- Provides visibility and observability for every packet.  
- Scales efficiently with large clusters.  

**Use Case:**  
Used in production-grade environments like Google GKE, AWS EKS, and high-security networks.

---

## 7. Packet Copy
In a normal data flow, when packets move between user space, kernel space, and the NIC, they’re often **copied** multiple times between memory buffers.  

**Flow Example:**
1. App writes data → user buffer.  
2. Kernel copies data → kernel buffer.  
3. NIC driver copies data again → DMA buffer.  

**Downside:**  
These extra copies consume CPU cycles and reduce throughput.

---

## 8. Zero Copy
**Zero Copy** avoids redundant copying of data between buffers by allowing direct data transfers between I/O devices and application buffers.

**System Calls Supporting Zero Copy:**
- `sendfile()` — Directly sends data from disk to network socket.  
- `splice()` — Moves data between file descriptors in kernel space.  

**Benefits:**
- Faster packet transmission.  
- Reduced CPU overhead.  
- Better throughput for high-performance servers (e.g., Nginx, Kafka).

---

## 9. User Space to Kernel Space Packet Route

When an application sends or receives network data, the packet travels between **user space** and **kernel space** through several layers of the operating system’s network stack.  
Here’s how the flow happens internally:

### **Outbound Packet Flow (When sending data)**

1. **User Application (User Space):**  
   The process begins when a program (like a web browser or ping utility) calls functions such as `send()` or `write()` to transmit data through a socket.

2. **System Call Transition:**  
   The data crosses the boundary from **user space** to **kernel space** via a system call.  
   At this point, the data is copied from the user buffer into a kernel buffer for processing.

3. **Transport Layer (Kernel Space):**  
   The kernel’s **TCP** or **UDP** protocol layer adds the necessary headers — handling tasks like segmentation, checksums, and port management.

4. **Network Layer:**  
   The **IP layer** adds IP headers, performs routing table lookups, and determines the next hop (destination MAC or gateway).

5. **Link Layer:**  
   The **Ethernet layer** encapsulates the packet inside an Ethernet frame and attaches MAC addresses.

6. **NIC Driver:**  
   The final frame is handed to the **Network Interface Card (NIC) driver**, which moves the packet into the NIC’s DMA buffer for physical transmission.

7. **Transmission:**  
   The **NIC** sends the packet over the physical medium (wired/wireless) to the destination host.

---

### **Inbound Packet Flow (When receiving data)**

1. **NIC Reception:**  
   The **NIC** receives an incoming packet from the network and performs basic validation (like CRC checks).  
   It then triggers an interrupt or uses **DMA** to transfer the packet data into kernel memory.

2. **NIC Driver:**  
   The driver notifies the kernel that a new packet has arrived and passes it to the network stack.

3. **Link Layer Processing:**  
   The kernel removes the Ethernet headers and identifies the encapsulated protocol (IPv4, IPv6, etc.).

4. **Network Layer Processing:**  
   The **IP layer** verifies the IP header, checks the destination IP, and passes the payload to the appropriate transport protocol (TCP/UDP).

5. **Transport Layer:**  
   The **TCP/UDP** layer reassembles segments if needed, validates checksums, and directs the payload to the correct socket based on port numbers.

6. **System Call Transition:**  
   The kernel copies the received data from its buffers into the application’s **user-space buffer** via a system call like `recv()` or `read()`.

7. **User Application:**  
   The application (e.g., browser or server) finally reads the data and processes it.

---

### **Summary**

- **Outbound path:** `User Space → System Call → Kernel Stack (TCP/IP) → NIC Driver → NIC → Network`  
- **Inbound path:** `Network → NIC → NIC Driver → Kernel Stack → System Call → User Space`

In high-performance environments, technologies like **Zero Copy**, **DPDK**, or **eBPF** are used to minimize these context switches and memory copies, improving throughput and reducing latency.

---

## 10. Libvirt
**Libvirt** is an open-source **API, daemon, and CLI tool** used to manage virtualization technologies like **KVM, QEMU, and Xen**.  

**Functions:**
- Create, configure, start, stop, and monitor VMs.  
- Manage **virtual networks, bridges, and storage pools**.  
- Abstracts underlying hypervisor commands into a consistent API.

**Why it matters:**  
It provides the foundation for tools like:
- `virt-manager` (GUI)  
- `virsh` (command-line tool)  
- Cloud stacks like **OpenStack Nova**  

Think of Libvirt as the “control center” for virtualization on Linux.

---

## 11. Nodes and Pods in Kubernetes (K8s)
Kubernetes (K8s) is a **container orchestration platform** that runs workloads across a cluster of machines.

### Node:
A **worker machine** (physical or virtual) that runs **pods**.  
Each node has:
- `kubelet` (agent)
- `container runtime` (e.g., containerd, CRI-O)
- `kube-proxy` (for networking)

### Pod:
The **smallest deployable unit** in Kubernetes — it can run one or more containers that share:
- **Network namespace**
- **Storage volumes**
- **Lifecycle**

Pods on different nodes communicate via the **CNI plugin** (like Flannel or Cilium), which provides the cluster network.

**Analogy:**  
If containers are "apps", then **pods are app instances**, and **nodes are servers** hosting them.

---

## Summary Table

| Concept | OSI Layer | Purpose | Common Use Case |
|----------|------------|----------|-----------------|
| **TUN** | L3 | Virtual IP routing | VPNs, software routers |
| **TAP** | L2 | Virtual Ethernet frames | VM bridges |
| **OVS** | L2/L3 | Virtual switching | SDN, VM networking |
| **VXLAN** | L2 over L3 | Overlay network tunneling | Cloud-scale networks |
| **Flannel** | L3 Overlay | Basic CNI for K8s | Simple pod networking |
| **Cilium** | Kernel (eBPF) | Advanced CNI + security | High-performance clusters |
| **Packet Copy** | — | Moves data between buffers | Causes CPU overhead |
| **Zero Copy** | — | Avoids redundant data movement | Improves throughput |
| **User ↔ Kernel Route** | — | Packet data path | Network optimization |
| **Libvirt** | — | VM & network management | KVM, QEMU |
| **Node/Pod** | — | Kubernetes structure | Cluster orchestration |

---