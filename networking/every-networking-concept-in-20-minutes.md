# Every networking concept, one app at a time

2026-11-06 · Source: [Every Networking Concept Explained In 20 Minutes](https://www.youtube.com/watch?v=xj_GjnD4uyI) by TechWorld with Nana

The video follows one made-up travel booking site, "Travel Buddy", from a single server to Kubernetes. Each concept shows up only when the app actually needs it, which is what makes it stick. These are my notes in my own words.

## 1. One server on the internet

| Problem | Concept | Analogy from the video |
|---|---|---|
| How do customers find the server? | **IP address**: a unique identifier so others can send it data (e.g. `203.0.113.10`) | A house address for mail |
| Nobody remembers numbers | **DNS**: turns `travelbuddy.com` into the IP | Phone contacts: tap "Mom", the phone finds the number |
| Website, DB and payments share one IP | **Ports**: numbered channels 1 to 65,535, one app per port | One building address, many apartment numbers |

Standard ports worth knowing by heart: `80` HTTP, `443` HTTPS, `3306` MySQL, `5432` Postgres. The demo payment service used a custom port, `9090`.

## 2. Growing up: segment, route, lock down

- **Network segmentation / subnets.** Once there are credit cards and personal data, one box running everything means one breach exposes everything. Split the network: front end in one subnet, app servers in another, databases in a third. (Hospital wings: maternity on one floor, surgery on another.)
- **Routing.** Subnets now need a path between them. A router decides how traffic gets from segment A to segment B. GPS for packets.
- **Firewalls.** Routing makes traffic *possible*; firewalls decide what is *allowed*.
  - *Host firewall* on one server: the DB accepts only `3306`, and only from the front-end subnet.
  - *Network firewall* between subnets: from the internet, allow only `80`/`443` into the front-end subnet and drop the rest.
  - The point is **layered security**. An attacker has to get past the network firewall *and* the host firewall.

## 3. Private servers that still need the internet: NAT

50 backend servers in a private subnet have private IPs (`10.0.0.25`, `10.0.0.26`, ...). Private IPs can't talk to the internet directly, but those servers still need OS updates and calls to external payment APIs. Giving each one a public IP costs money and exposes them.

**NAT (network address translation)** lets many private hosts share one public IP for *outbound* traffic:
1. `10.0.0.25` sends its request to the NAT device.
2. NAT swaps the private source address for its own public IP and sends the request out.
3. The reply comes back to NAT, which remembers which host asked and forwards it.

Analogy: the office receptionist places outside calls on the company's main line and routes the answer back to the right desk. The servers can reach out, but nobody outside can reach in.

## 4. Moving to the cloud: same ideas, managed for you

Hardware takes weeks to buy and rack, and capacity has to be guessed months ahead. The cloud rents capacity by the minute. **The concepts don't change; the cloud sells them as managed services.**

| On-prem | AWS-style cloud |
|---|---|
| Your own network | **VPC**: your isolated slice of the provider's network (your own floor in a shared office tower) |
| Subnets | Public subnets (internet-facing) and private subnets (protected) |
| Edge router to the internet | **Internet gateway** for the public subnets |
| Routing config | **Route tables**, one per subnet |
| NAT box | **NAT gateway**: sits in a public subnet, and private subnets send outbound traffic through it |
| Firewalls | Security groups |

## 5. Containers: Docker networking

Microservices bring "works on my laptop, breaks in prod". A container packs code, runtime, libraries and config into one portable unit (a food truck vs. a restaurant you'd have to rebuild to move).

- **Bridge network**: a private network that only exists on one host. Containers on the same bridge reach each other **by container name**.
- **Port mapping**: the app inside the container listens on its own port. `docker run -p 9090:9090 --name payment ...` forwards host port 9090 to container port 9090. It's NAT again: translating addresses and ports between two networks.
- **Overlay network**: spans many hosts, so containers on different servers behave as if they're on one network.

## 6. Kubernetes networking

Hundreds of containers across dozens of servers raise new questions. Where does a new one go? Who notices a crash? How do services find each other when containers keep moving? Kubernetes is the automated building manager.

- **Pod**: the smallest unit, one or more containers (usually one). **Each pod gets its own IP**, and the containers in it share that IP (one apartment, one address).
- **Pods are ephemeral.** Every recreate (crash, rollout) gets a new IP, so never hard-code pod IPs.
- **Service**: a stable IP plus a DNS name (e.g. `database-service`) in front of a changing set of healthy pods. Clients talk to the service and never notice pods being replaced. (A department phone number: the person answering changes, the number doesn't.)
- **Ingress**: a single entry point for outside traffic, routing by host or path to the right service, for example:
  - `travelbuddy.com` → website service
  - `travelbuddy.com/api/booking` → booking service
  - `travelbuddy.com/api/payment` → payment service

## The 5 fundamentals (the video's recap)

1. **IP + DNS**: everything needs an address, and names resolve to addresses.
2. **Ports**: many apps on one host each listen on their own port.
3. **Subnets + routing**: segment the network, then connect the segments.
4. **Firewalls**: control what's allowed between segments and to which ports.
5. **NAT**: private hosts reach the internet through a shared public IP.

The tools change (physical routers → VPCs, hardware firewalls → security groups), but the concepts stay the same.

## How I'd use it

When something "can't connect", walk the same ladder top to bottom: does the name resolve (`dig`), is anything listening on the port (`ss -tlnp`), is there a route (`ip route`, `traceroute`), and is a firewall, security group or NetworkPolicy dropping it? In Kubernetes, always point clients at the Service DNS name, never at a pod IP.
