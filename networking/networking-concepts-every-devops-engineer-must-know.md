# Networking from physical servers to Kubernetes

2026-11-06 · Source: [Networking Concepts Every DevOps Engineer Must Know](https://www.youtube.com/watch?v=w0SQGCt-6Ro) by TechWorld with Nana

This one traces a made-up e-commerce shop, "ShopEasy", through four eras: physical server → VMs → cloud → containers → Kubernetes. It goes deeper than the 20-minute video on network interfaces, hypervisors, CNI, kube-proxy and NetworkPolicies. My notes, in my own words.

Chapters: [Physical server](https://www.youtube.com/watch?v=w0SQGCt-6Ro&t=58s) · [VMs and hypervisors](https://www.youtube.com/watch?v=w0SQGCt-6Ro&t=630s) · [Cloud / AWS VPC](https://www.youtube.com/watch?v=w0SQGCt-6Ro&t=1090s) · [Docker](https://www.youtube.com/watch?v=w0SQGCt-6Ro&t=1353s) · [Kubernetes](https://www.youtube.com/watch?v=w0SQGCt-6Ro&t=1696s) · [Takeaways](https://www.youtube.com/watch?v=w0SQGCt-6Ro&t=2231s)

## Phase 1: The physical server era

**IP address.** A unique address for a device on a network, like a street address.
- **Network**: a group of devices that can talk directly to each other (houses in one neighbourhood).
- **IPv4**: four numbers from 0 to 255, e.g. `203.0.113.10`. Part of the address identifies the *network* (the street) and the rest identifies the *host* (the house number).
  - *My addition:* the video simplifies this to "first three parts = network", which is true for a `/24`. In practice the split comes from the subnet mask / CIDR prefix (`/16`, `/24`, `/28`, ...).

**Routers / default gateway.** To reach a device on *another* network, a host sends traffic to its default gateway. Routers forward it hop by hop using the network part of the destination IP, like post offices sorting mail between neighbourhoods.

**DNS.** The internet's address book: `shopeasy.com` → IP. The browser asks a DNS server, which answers in milliseconds. A bonus: the IP behind the name can change and users still land on the right server.

**Network interfaces.** The "front doors" of a server where all traffic enters and leaves.
- `eth0`: the main wired interface (the front door)
- `eth1`: another wired interface
- `wlan0`: Wi-Fi
- `lo`: loopback, for the machine talking to itself (`localhost`)

**Ports.** The "rooms" inside the house: web on `80`/`443`, DB on `3306`/`5432`. The **OS is the house manager**: it reads the destination port on each packet and hands it to the program listening there.

**Why a single physical server hurts**
- Single point of failure: if the box dies, the business is offline.
- Can't scale for peak (holiday) traffic.
- You pay for it even when it sits idle.
- Every upgrade means downtime.

## Phase 2: Virtual machines and hypervisors

Virtualization splits one big house into self-contained **apartments**.
- A **VM** is a full computer made in software: virtual CPU, memory, disk and NIC, and its own guest OS.
- VMs are **isolated**, so a crash in one doesn't touch the others. Resources are used better, too.
- The big win is **portability**: a VM's whole state can be **snapshotted** and restored on a different physical host if the original dies.

Networking inside the box:
- The **hypervisor** sits between the hardware and the VMs. It shares out the real CPU, RAM and NIC and builds a **virtual network**, the "invisible cables" between VMs.
- Each VM gets a **private IP** on that internal network (web `10.0.0.5`, DB `10.0.0.6`, cache `10.0.0.7`). VMs talk to each other over these without touching the public internet.
- Each virtual interface has a **firewall** checking sender, target port and whether the traffic is allowed.
- Traffic to the outside world leaves through the host's public IP via **NAT**.

## Phase 3: Cloud networking (AWS)

The cloud is "renting apartments in a huge, professionally managed skyscraper". Providers built the data centres and wiring and sell slices through an API or console.

| Concept | What it is | Analogy |
|---|---|---|
| **VPC** | Your isolated private network in the cloud | Your private wing of the tower |
| **Public subnet** | Reachable from the internet, for web servers | Storefront facing the street |
| **Private subnet** | Not directly exposed, for databases and internal apps | Back office deep inside |
| **Security group** | Fine-grained, instance-level firewall rules | Guard at each door with a guest list |
| **Route table** | Where traffic from a subnet goes next | Building directory |
| **NAT gateway** | Lets private subnets make outbound calls (updates, APIs) while staying unreachable from outside | Trusted concierge who fetches things but never gives out your office address |

## Phase 4: Docker networking

The monolith ("one giant factory") breaks into **microservices** (small workshops). **Containers** are the fully stocked toolbox for each workshop: code, dependencies and config travel together.

**Containers vs VMs.** Containers share the host's (or VM's) **kernel** and only isolate the app layer. That makes them start fast and keeps images in **MBs**, while VMs are in **GBs**.

| Docker network | What it does |
|---|---|
| **Bridge** (default) | Private network on one host. Containers get internal IPs, and Docker's built-in DNS lets them call each other **by name** (`api`, `redis`). |
| **Overlay** | Encrypted network across many hosts, so containers on different VMs talk as if they were local. |
| **Port mapping** | `docker run -p 80:8080 webapp` sends host port 80 to container port 8080 (the lobby receptionist transferring calls to an extension). |

## Phase 5: Kubernetes networking

Kubernetes is the city planner: it places containers, restarts failed ones and scales with demand.

- **Pod IPs, flat network.** Every pod gets its own IP. Any pod can reach any other pod directly, across nodes, with **no NAT or port mapping**. It works like one flat company LAN.
- **CNI (Container Network Interface).** The "contractor that wires each new room". When a pod is scheduled, Kubernetes calls the CNI plugin, which:
  1. creates a virtual ethernet interface (a veth pair) for the pod,
  2. assigns it a unique IP,
  3. sets up routes so the pod is reachable from anywhere in the cluster.
- **Service.** Pod IPs change all the time, so a Service gives a **stable virtual IP**, like a company hotline. **kube-proxy** programs routing rules (iptables/IPVS) that load-balance calls to healthy pods behind it.
- **Cluster DNS.** Every Service registers a name (`backend-service`). The front end just calls the name.
- **Ingress.** One managed front door for outside traffic. It routes by **host or path** to services, can terminate TLS and enforce access, and doesn't expose every internal room. An **Ingress controller** does the actual work.
- **NetworkPolicy.** By default **all pods can talk to all pods** (every door unlocked). NetworkPolicies are door locks: allow/deny **ingress and egress** by pod labels, namespaces or IP ranges.
  - Example: only pods labelled `app: api-server` may reach the DB pods on `5432`; everything else is blocked.
  - Enforced by the CNI plugin (e.g. via iptables). *My addition:* a CNI that doesn't support NetworkPolicy silently ignores them, so check your plugin.

## Takeaways

The fundamentals never changed. The tools just got more powerful (and more complex):
- **Addresses**: everything needs an IP.
- **Rules**: firewalls → security groups → NetworkPolicies.
- **Names**: DNS → service discovery.
- **Routes**: routing tables decide where traffic goes.

Every abstraction layer adds capability *and* complexity. A solid grip on the basics is what lets you troubleshoot confidently at any layer.

## How I'd use it

When debugging a service that can't reach another in Kubernetes, I check each layer this video describes in order: DNS (`nslookup backend-service` from a debug pod), Service → endpoints (`kubectl get endpoints`), whether a NetworkPolicy is blocking the path (`kubectl get networkpolicy -A`), and finally pod-to-pod reachability across nodes, which points at CNI or routes.
