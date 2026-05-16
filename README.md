# 🌐 Dual-WAN Edge: PCC Load Balancing + Recursive Route Failover
### *My Second Project as a Network Engineer*

> Don't trust an ISP just because the cable is plugged in.

---

## 🗺️ The Topology

![Network Topology](https://YOUR-IMAGE-HOST.com/topology.png)

**Edge Layer:** Dual WAN Connections (ISP1 on Eth1, ISP2 on Eth2)  
**Core Gateway:** Core Router handling Policy-Based Routing (PBR)  
**Failover Stack:** Connection Marking + Routing Tables + Multi-Hop ICMP Recursive Probes  
**Segmentation:** Isolated local subnets via Pre-Routing Mangle Rules

---

## 🎭 The Lore

Okay so here's the scenario:

Most multi-WAN networks run basic failover or simple static routes. The problem? Standard routing only checks if the physical cable to your ISP modem is plugged in.

That's it. Just the cable.

If the ISP has a massive backhaul outage deeper in their network but their modem stays powered on? Your router has no idea. It keeps sending data into a black hole. Users think the internet is down. Chaos.

For this project, I wanted two things:

1. **Both ISPs working at the same time** – No idle backup wasting bandwidth
2. **Real health checks** – Not just "is the cable plugged in?"

So I built a Dual-WAN architecture where:
- Traffic splits 50/50 across both ISPs
- Each ISP gets checked past the modem—all the way to the actual internet
- If one dies, traffic shifts instantly

Every megabit of bandwidth earns its keep.

---

## 🛠️ Performance Highlights

**Policy-Based Mangle Architecture**  
Developed a structured mangle pipeline that separates local subnet exceptions from outbound traffic. Internal routing takes priority before any WAN load-balancing happens.

**Symmetric Traffic Splitting (PCC)**  
Engineered a 2-stream Per-Connection Classifier splitting traffic cleanly down Remainder 0 (ISP1) and Remainder 1 (ISP2). Effectively doubled edge backplane utilization.

**Upstream Blackhole Elimination**  
Deployed independent virtual routing tables (ISP1 and ISP2) mapped to multi-hop recursive paths. Replaced interface-dependent failover with active, off-net ICMP path tracking.

**Inbound State Alignment**  
Configured input chain tracking to catch new incoming connections on Eth1 and Eth2. Locked output routing marks to the matching interface. Preserved external access states.

![PCC Load Balancing Demo](https://github.com/user-attachments/assets/0a27f1f6-234f-4fe4-9e7e-617283c09e23)

---

## 🧪 The Proof: Validation Tests

### Test 1: The Upstream Interdiction (Recursive Failover)

**What I did:** Simulated a backhaul ISP outage by disconnecting the upstream fiber link past the ISP-1 edge router. Kept the local physical link on Eth1 completely active.

**What happened:** Standard static routing kept the interface active. But the recursive engine immediately noticed 1.1.1.1 stopped responding to ICMP probes. Within 3 dropped pings, Table ISP1 dynamically deactivated the dead primary route.

**The win:** Because of backup routes built into the specialized tables (Distance=2), traffic mapped to the ISP1 table instantly rerouted out of Eth2. No waiting for the physical interface to go down.

![Recursive Failover Test](https://YOUR-IMAGE-HOST.com/recursive-failover.gif)

---

### Test 2: Bandwidth Maximization (PCC Balancing)

**What I did:** Launched multiple high-bandwidth download streams across different internal client machines.

**What happened:** Monitored real-time traffic graphs. Traffic split evenly across Eth1 and Eth2. Both connections handling data simultaneously. No idle link.

![PCC Bandwidth Split](https://YOUR-IMAGE-HOST.com/pcc-balancing.gif)

---

### Test 3: Symmetrical Inbound Reply (State Tracking)

**What I did:** Sent external remote-management probes directly to the WAN IP of Eth2 while Eth1 was designated as the primary routing table path.

**What happened:** Router successfully processed the connection tracking rules. Reply packets bypassed the default routing table and exited directly back out of Eth2. No upstream carrier drop-offs from asymmetric path mismatches.

![Inbound State Tracking](https://YOUR-IMAGE-HOST.com/symmetric-reply.png)

---

### Test 4: The Local Isolation Layer (Mangle Acceptance)

**What I did:** Executed an internal inter-subnet file transfer from VLAN 10 to VLAN 20 while running continuous WAN balancing.

**What happened:** Packet counters on the initial Pre-Routing Mangle chain hit the accept rule instantly. Traffic remained internal at line-rate speed. 0% of local packets pushed out to internet interfaces.

![Local Isolation Test](https://YOUR-IMAGE-HOST.com/local-traffic.png)

---

## 🚧 Engineering Challenges

**The Mangle Pipeline Logic (passthrough rules)**

Managing packet markings can get messy fast. If a routing mark overwrites a connection mark prematurely, the load balancing logic completely breaks down.

**How I solved it:** Enforced strict architectural rules inside the RouterOS Mangle table. Set `passthrough=YES` on all connection-marking rules so packets could continue down the chain to be evaluated. Explicitly declared `passthrough=NO` the moment a final routing table mark (ISP1 or ISP2) was stamped. Stopped unnecessary processing. Sped up CPU cycles.

**Recursive Scope Deadlocks**

During initial recursive setup, the router can enter a routing loop. It tries to look up the monitor target using the target itself. The route flaps or goes invalid.

**How I solved it:** Overrode the default routing scopes to create a clean logical hierarchy. Configured virtual monitoring routes (0.0.0.0/0 via 1.1.1.1) to look deep with Target Scope=30. Anchored the actual physical next-hop gateway (1.1.1.1 via ISP Edge IP) strictly to Scope=10. This lets the recursive engine safely resolve the virtual target over the concrete physical link.

---

## 💡 Final Thoughts

Edge architecture shouldn't trust an ISP blindly.

Setting up basic active-passive failover where a secondary internet link sits dark? That's a junior configuration. It leaves bandwidth on the table. It fails when upstream networks break silently.

Active-Active Dual-WAN with recursive checking is **real engineering**. It forces you to:
- Manipulate routing tables directly
- Manage packet markings at granular level
- Build an edge that dynamically assesses network health based on actual path reality

**The Metric That Matters: Dynamic Gateways**

I audited the active flags in the WinBox routing list during a simulated backhaul drop. The recursive engine kept data moving. Users saw 100% uptime regardless of provider instability.

**Why This Matters to an Employer:**

Most engineers stop at "does the link work?" I ask "does the path work?"

Physical link up doesn't mean internet up. I've seen ISPs stay "connected" while dropping every packet past their first hop.

My edge doesn't trust lights. It trusts the actual route to 1.1.1.1. If that dies, traffic moves. No black holes. No user complaints.

That's the difference between availability and reliability.

---

*Second project down. More to come.* 🔥
