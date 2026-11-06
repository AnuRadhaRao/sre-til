# Networking fundamentals: combined summary

2026-11-06 · Sources: [Every Networking Concept Explained In 20 Minutes](https://www.youtube.com/watch?v=xj_GjnD4uyI) and [Networking Concepts Every DevOps Engineer Must Know](https://www.youtube.com/watch?v=w0SQGCt-6Ro), both by TechWorld with Nana

Both videos tell the same story: one app grows from a single server to Kubernetes, and a new networking concept appears every time it hits a wall. This page merges them into one cheat sheet. Detailed notes: [video 1](every-networking-concept-in-20-minutes.md), [video 2](networking-concepts-every-devops-engineer-must-know.md).

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

## The journey in eight steps

1. **IP address.** Every device needs a unique address. IPv4 is four numbers from 0 to 255, split into a network part and a host part by the CIDR prefix.
2. **DNS.** Humans use names, so DNS resolves them to IPs. The IP can change behind the name.
3. **Interfaces and ports.** Traffic enters through an interface (`eth0`, `lo`, ...). The OS hands each packet to the process listening on its destination port (80/443 web, 3306 MySQL, 5432 Postgres).
4. **Subnets + routing.** Split the network so one breach doesn't expose everything, then let routers (the default gateway) connect the segments.
5. **Firewalls, layered.** Routing makes traffic possible and firewalls decide what is allowed. Use network firewalls *and* host firewalls, so an attacker has to beat both.
6. **NAT.** Private hosts share one public IP for outbound calls, and nobody outside can start a connection in. Port mapping in Docker is the same trick.
7. **Virtualization → cloud.** The hypervisor builds virtual networks between VMs, and snapshots make VMs portable. The cloud packages it all as managed services: VPC, public/private subnets, route tables, internet gateway, NAT gateway, security groups.
8. **Containers → Kubernetes.**
   - Docker: a **bridge** on one host (reach containers by name), **overlay** across hosts, and **port mapping** to get outside traffic in.
   - Kubernetes: every **pod gets its own IP** via **CNI** on a flat network, with no NAT between pods.
   - Pods are ephemeral, so a **Service** gives a stable IP and DNS name, and **kube-proxy** load-balances to healthy pods.
   - **Ingress** is the single smart front door, routing by host and path.
   - **NetworkPolicy** locks the doors, because by default every pod can reach every pod.

## Analogies that make it stick

| Concept | Analogy |
|---|---|
| IP address | Street address of a house |
| DNS | Phone contacts / the internet's address book |
| Interface (`eth0`) | The front door |
| Port | Apartment or room number inside the building |
| Subnet | Hospital wings / floors |
| Router | GPS / post office sorting mail |
| Firewall / security group | Security guard with a guest list |
| NAT | Receptionist placing calls on the company's main line |
| VPC | Your private floor in a shared office tower |
| Container | Food truck / fully stocked toolbox |
| Pod | Apartment unit sharing one address |
| Service | Department hotline: the people change, the number doesn't |
| Ingress | Reception desk sending visitors to the right department |
| NetworkPolicy | Door locks and role-based badges |

## SRE troubleshooting checklist (my addition)

The same layers make a debugging ladder. Work through it top to bottom:

1. **Name**: does it resolve? `dig app.example.com`, or `nslookup my-svc` from inside a pod.
2. **Listener**: is anything listening on the expected port? `ss -tlnp`, `kubectl get endpoints my-svc`.
3. **Route**: is there a path? `ip route`, `traceroute`, route tables, internet/NAT gateway for private subnets.
4. **Rules**: is something dropping it? Host firewall, security group/NACL, NetworkPolicy.
5. **Translation**: is NAT or port mapping right? Check the `-p host:container` order, Service `port` vs `targetPort`, and Ingress host/path rules.

## How I'd use it

This is my mental model for interviews and incidents. When asked "what happens when you curl a service in Kubernetes", I walk the table above left to right. When a call fails, I walk the checklist: name, listener, route, rules, translation.
