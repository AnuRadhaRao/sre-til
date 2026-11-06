# Networking concepts

2026-11-06

My study notes on networking fundamentals, from a single physical server all the way to Kubernetes. Each concept shows up at the point an app actually needs it. Sources are listed under [References](#references).

## The one idea to remember

**The concepts never change. Only the implementation does.**

| Concept | Physical / on-prem | VMs | Cloud (AWS) | Docker | Kubernetes |
|---|---|---|---|---|---|
| **Address** | Public IP on `eth0` | Private IP from the hypervisor's virtual network | Private IPs in a VPC subnet | Container IP on a bridge | Pod IP (assigned by CNI) |
| **Name → address** | DNS | DNS | Route 53 / VPC DNS | Docker's built-in DNS (container names) | CoreDNS + Service names |
| **Which app?** | Ports, chosen by the OS | Ports per VM | Ports + listeners | Port mapping `-p host:container` | containerPort / Service port |
| **Segmentation** | Subnets | Virtual networks | VPC + public/private subnets | Separate networks | Namespaces (+ policies) |
| **Path between segments** | Routers / default gateway | Hypervisor routing | Route tables, internet gateway | Bridge / overlay network | Flat pod network (CNI routes) |
| **Allow / deny** | Host + network firewalls | Per-vNIC firewalls | Security groups (NACLs too) | Not much by default | NetworkPolicy |
| **Private → internet** | NAT device | NAT on the host | NAT gateway | Host NAT (masquerade) | Node SNAT / egress gateway |
| **Stable entry for clients** | DNS → server | DNS → host | Load balancer | Published port | Service (stable VIP) + Ingress |

---

## 1. One physical server

### IP address
A unique address for a device on a network, like a house's street address.
- A **network** is a group of devices that can talk directly to each other (houses in one neighbourhood).
- **IPv4** is four numbers from 0 to 255, e.g. `203.0.113.10`. Part of the address identifies the *network* (the street) and the rest identifies the *host* (the house number). The **CIDR prefix** (`/16`, `/24`, `/28`) sets where that split falls, so "first three numbers = network" is only true for a `/24`.

### Routers and the default gateway
To reach a device on *another* network, a host sends traffic to its **default gateway**. Routers forward it hop by hop using the network part of the destination IP, like post offices sorting mail between neighbourhoods.

### DNS
The internet's address book: it turns `shop.com` into an IP. The browser asks a DNS server and gets an answer in milliseconds. The IP behind a name can change and users still land on the right server. (Phone contacts: you tap "Mom", the phone finds the number.)

### Network interfaces
The "front doors" where all traffic enters and leaves a server.

| Interface | What it is |
|---|---|
| `eth0` | Main wired interface (the front door) |
| `eth1` | Another wired interface |
| `wlan0` | Wi-Fi |
| `lo` | Loopback, for the machine talking to itself (`localhost`) |

### Ports
Ports are numbered channels from 1 to 65,535, and each app listens on its own. One building address (the IP), many apartment numbers (the ports). The **OS is the house manager**: it reads each packet's destination port and hands the packet to the program listening there.

Ports to know by heart: `80` HTTP, `443` HTTPS, `3306` MySQL, `5432` Postgres.

### Why one server hurts
- **Single point of failure**: if the box dies, the business is offline.
- **Can't scale** for peak (holiday) traffic.
- **Paying for idle** capacity.
- **Downtime** for every upgrade.

## 2. Segment, route, lock down

### Subnets (network segmentation)
Once you hold credit cards and personal data, one box running everything means one breach exposes everything. Split the network: front end, app servers and databases each go in their own subnet. (Hospital wings: maternity on one floor, surgery on another.)

### Routing
Subnets need a path between them. A router decides how traffic gets from segment A to segment B. It's GPS for packets.

### Firewalls, layered
Routing makes traffic *possible*. Firewalls decide what is *allowed*.
- **Host firewall** on one server: the database accepts only `3306`, and only from the front-end subnet.
- **Network firewall** between subnets: from the internet, allow only `80`/`443` into the front-end subnet and drop everything else.
- An attacker has to beat **both** layers. Security is always layered.

### NAT (network address translation)
Backend servers in a private subnet have private IPs (`10.0.0.25`, `10.0.0.26`, ...) that can't reach the internet directly, but they still need OS updates and calls to external APIs. Giving each one a public IP costs money and exposes it. NAT lets many private hosts **share one public IP for outbound traffic**:
1. `10.0.0.25` sends its request to the NAT device.
2. NAT swaps the private source address for its own public IP and sends the request out.
3. The reply comes back to NAT, which remembers which host asked and forwards it.

The servers can reach out, but nobody outside can start a connection in. (A receptionist places outside calls on the company's main line and routes the answer back to the right desk.)

## 3. Virtual machines and hypervisors

Virtualization splits one big house into self-contained **apartments**.
- A **VM** is a full computer made in software: virtual CPU, memory, disk and NIC, plus its own guest OS.
- VMs are **isolated**, so a crash in one doesn't affect the others. Hardware is used better.
- The big win is **portability**: a VM's whole state can be **snapshotted** and restored on a different physical host if the original dies.

Networking inside the box:
- The **hypervisor** sits between the hardware and the VMs. It shares out the real CPU, RAM and NIC and builds a **virtual network**, the "invisible cables" between VMs.
- Each VM gets a **private IP** on that network (web `10.0.0.5`, DB `10.0.0.6`, cache `10.0.0.7`). VMs talk to each other without touching the public internet.
- Each virtual interface has a **firewall** that checks the sender, the target port and whether the traffic is allowed.
- Outbound traffic leaves through the host's public IP via **NAT**.

## 4. Cloud networking (AWS)

Buying and racking hardware takes weeks, and capacity has to be guessed months ahead. The cloud rents capacity by the minute from providers who already built the data centres. **The same concepts are sold as managed services.**

| Concept | What it is | Analogy |
|---|---|---|
| **VPC** | Your isolated private network in the provider's cloud | Your own floor or wing in a shared office tower |
| **Public subnet** | Reachable from the internet, for web servers | Storefront facing the street |
| **Private subnet** | Not directly exposed, for databases and internal apps | Back office deep inside |
| **Internet gateway** | Connects public subnets to the internet | Main entrance |
| **Route table** | One per subnet, says where traffic goes next | Building directory / signposts |
| **NAT gateway** | Sits in a public subnet so private subnets can make outbound calls while staying unreachable | Concierge who fetches things but never gives out your office address |
| **Security group** | Instance-level firewall rules | Guard at each door with a guest list |

## 5. Containers and Docker networking

The monolith ("one giant factory") breaks into **microservices** (small workshops), and "works on my laptop, breaks in prod" becomes a daily problem. A **container** packs code, runtime, libraries and config into one portable unit, like a food truck instead of a restaurant you'd have to rebuild to move.

**Containers vs VMs.** Containers share the host's (or VM's) **kernel** and only isolate the app layer. They start fast, and images are measured in **MBs**, while VMs are in **GBs**.

| Docker networking | What it does |
|---|---|
| **Bridge** (default) | Private network on one host. Containers get internal IPs, and Docker's built-in DNS lets them reach each other **by name** (`api`, `redis`). |
| **Port mapping** | `docker run -p 80:8080 webapp` sends host port 80 to container port 8080. This is NAT again, translating ports between two networks. (The lobby receptionist transferring calls to an extension.) |
| **Overlay** | Encrypted network across many hosts, so containers on different servers talk as if they were local. |

## 6. Kubernetes networking

Hundreds of containers across dozens of servers raise new questions. Where does a new one go? Who notices a crash? How do services find each other when containers keep moving? Kubernetes is the automated building manager: it places containers, restarts failed ones and scales with demand.

### Pods
The smallest unit is the **pod**: one or more containers (usually one) sharing **one IP** (one apartment, one address).
- **Flat network**: any pod can reach any other pod directly across nodes, with **no NAT or port mapping**. It behaves like one big company LAN.
- **Pods are ephemeral**: every recreate (crash, rollout) gets a new IP, so never hard-code pod IPs.

### CNI (Container Network Interface)
CNI is the "contractor that wires each new room". When a pod is scheduled, Kubernetes calls the CNI plugin, which:
1. creates a virtual ethernet interface (a veth pair) for the pod,
2. assigns it a unique IP,
3. sets up routes so the pod is reachable from anywhere in the cluster.

### Services and kube-proxy
A **Service** gives a **stable virtual IP and DNS name** (e.g. `backend-service`) in front of a changing set of healthy pods. **kube-proxy** programs routing rules (iptables/IPVS) to load-balance across those pods. Clients call the name and never notice pods being replaced. (A department hotline: the person answering changes, the number doesn't.)

### Cluster DNS
Every Service registers its name in the cluster's internal phone book, so the front end just calls `backend-service`.

### Ingress
One managed front door for outside traffic. It routes by **host or path** to the right Service, can terminate TLS and enforce access, and doesn't expose every internal room. An **Ingress controller** does the actual work. Example:
- `shop.com` → website service
- `shop.com/api/booking` → booking service
- `shop.com/api/payment` → payment service

### NetworkPolicy
By default **every pod can talk to every pod** (every door unlocked). NetworkPolicies are door locks and role-based badges: allow or deny **ingress and egress** by pod labels, namespaces or IP ranges.
- Example: only pods labelled `app: api-server` may reach the DB pods on `5432`. Everything else is blocked.
- The CNI plugin enforces these rules. A CNI that doesn't support NetworkPolicy **silently ignores** them, so check your plugin.

---

## The fundamentals in one list

1. **Addresses + names**: everything needs an IP, and DNS resolves names to IPs.
2. **Ports**: many apps on one host each listen on their own port.
3. **Subnets + routing**: segment the network, then connect the segments.
4. **Rules**: firewalls → security groups → NetworkPolicies decide what's allowed.
5. **NAT**: private hosts reach the internet through a shared public IP.

Each abstraction layer adds capability *and* complexity. Knowing the basics is what lets you troubleshoot confidently at any layer.

## Analogies that make it stick

| Concept | Analogy |
|---|---|
| IP address | Street address of a house |
| DNS | Phone contacts / the internet's address book |
| Interface (`eth0`) | The front door |
| Port | Apartment or room number |
| Subnet | Hospital wings / floors |
| Router | GPS / post office sorting mail |
| Firewall / security group | Security guard with a guest list |
| NAT | Receptionist placing calls on the company's main line |
| VPC | Your private floor in a shared office tower |
| Container | Food truck / fully stocked toolbox |
| Pod | Apartment unit sharing one address |
| Service | Department hotline |
| Ingress | Reception desk sending visitors to the right department |
| NetworkPolicy | Door locks and role-based badges |

## SRE troubleshooting checklist

The same layers make a debugging ladder. When something "can't connect", work through it top to bottom:

1. **Name**: does it resolve? `dig app.example.com`, or `nslookup my-svc` from inside a pod.
2. **Listener**: is anything listening on the expected port? `ss -tlnp`, `kubectl get endpoints my-svc`.
3. **Route**: is there a path? `ip route`, `traceroute`, the subnet route tables, an internet or NAT gateway for private subnets.
4. **Rules**: is something dropping it? Host firewall, security group/NACL, `kubectl get networkpolicy -A`.
5. **Translation**: is NAT or port mapping right? Check the `-p host:container` order, Service `port` vs `targetPort`, and Ingress host/path rules.

## References

- [Every Networking Concept Explained In 20 Minutes](https://www.youtube.com/watch?v=xj_GjnD4uyI), TechWorld with Nana
- [Networking Concepts Every DevOps Engineer Must Know](https://www.youtube.com/watch?v=w0SQGCt-6Ro), TechWorld with Nana
