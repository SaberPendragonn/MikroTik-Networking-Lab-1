

https://github.com/user-attachments/assets/3349bb00-54f2-491d-a717-14324b2cf89e



# Dual-WAN Enterprise Gateway: PCC Load Balancing & Recursive Routing Failover
### *Engineered a Per-Connection Classifier (PCC) Load Balancing fabric with Layer 7 Session Stickiness and Recursive Gateway for Multi-ISP High-Availability.*

> My Second Project

---

## The Lore

Okay so here's the scenario:

Standard Dual-WAN setups often suffer from two critical flaws: Asymmetric Session Breaks and "Ghost" Gateway Failures.

If a router uses simple Round-Robin packet distribution like the default PFIFO, secure websites (HTTPS/Banking) will drop the session because the source IP appears to flip-flop every second.

Furthermore, if the ISP's physical modem stays "Up" but their internal network loses internet connectivity, a standard router remains "blind" to the failure and continues sending packets into a black hole.

I engineered a solution that ensures both LOad Blancing and Intelligent Recursive Self-Healing:

1. **PCC Hashing & Stickiness:** I utilized the PCC matcher to hash traffic based on source/destination IPs. This ensures that once a session is established on ISP 1 or ISP 2, it "sticks" there, preserving stateful connections while allowing different client connections to utilize the full capacity of both ISPs.

2. **Recursive Monitoring:** I decoupled the route "Check-Gateway" from the ISP's local IP. By routing toward a virtual target (like Google DNS & CloudFlare) via the specific ISP gateway, the router only considers the path "Active" if it can reach the actual internet. If the path fails, the routing table dynamically pivots all traffic to the healthy ISP.

---

## Performance Highlights

**Granular Multi-WAN Hashing**  
Deployed a complete PCC engine capable of tracking separate network streams and dynamically sharing the transport load based on client transport definitions.

**Failover Verification**  
Constructed a multi-hop recursive routing path that targets global web nodes such as 8.8.8.8 and 1.1.1.1 to prevent gateway blackholing during upstream carrier outages.

**Session-State**  
Anchored connection tracking boundaries across the Mangle table to prevent multi-WAN packet reordering and keep secure user connections stable.

---

## The Proof

### Test 1: Multi-Client PCC Hashing Verification

**What I did:** Monitored the IP Mangle table and Connection Tracker while initiating traffics from my Kali Linux and Windows 10 clients running on VLAN 10 and VLAN 20.

**What happened:** The PCC hashing engine successfully identified the unique source/destination address. Connection Marks inside the WinBox tracking confirmed that the VLAN 10 client was hashed to ISP 1, while the VLAN 20 client was concurrently hashed to ISP 2, proving successful load distribution across the multi-WAN fabric. Note that one session opens multiple connections so it is normal for one IP to hash to different ISPs.

https://github.com/user-attachments/assets/d7eb6e7e-eb9b-46ef-8715-504ab217f935

---

### Test 2: Public IP Hashing & Session Stickiness

**What I did:** Opened multiple independent browser sessions on a single host to query Public IP discovery websites and repeatedly refreshed the pages to verify session persistence.

**What happened:** Different browser tabs successfully hashed to different ISPs (Tab A showed ISP 1's public IP, Tab B showed ISP 2's public IP). Despite multiple heavy refreshes, each specific session remained pinned to its respective ISP, confirming that PCC prevents session-reset issues.


https://github.com/user-attachments/assets/e8a3f087-f044-4a2f-a748-1ebdb17bdb49


---

### Test 3: ECMP vs. PCC - Debu

https://github.com/user-attachments/assets/88eb263d-1e32-409d-9b0d-9b2196860f04

nking the Round-Robin Myth and why PCC is better!

**What I did:** Switched the core routing configuration to Equal-Cost Multi-Path (ECMP) to analyze the differences in traffic hashing under stress compared to PCC.

**What happened:** Testing successfully debunked a common networking myth. Many assume ECMP sends packets via simple, alternating round-robin distribution which breaks active sessions on a refresh. In reality, once an ECMP connection is established, it hashes and sticks to that single ISP. When the webpage was refreshed, the session remained completely sticky on its path, proving ECMP keeps connection stickiness intact but lacks the granular control over specific traffic profiles that PCC provides through Mangle rules.

**ECMP:**



**PCC:**



---

### Test 4: Recursive Route Failover & Convergence Timing

**What I did:** Simulated a "Ghost" gateway failure by creating an explicit packet drop rule for the recursive targets (8.8.8.8 and 1.1.1.1) on ISP 1 while keeping the physical gateway interface active, using a high-frequency ping (0.1s interval) to measure the exact failover time.

**What happened:** Without recursive routing, the connections became confused and packets were dropped into an active black hole because the router thought the up-state interface was healthy.

**WITHOUT RECURSIVE ROUTING:**



**WITH RECURSIVE ROUTING:**



With recursive routing active, the router detected the loss of 8.8.8.8/1.1.1.1 reachability. Traceroute logs showed the path initially exiting via ISP 1, then dynamically shifting to ISP 2 the moment the recursive check failed. The high-frequency ping recorded 340 dropped packets and 56 received during the transition. This calculated to a total failover convergence time of exactly **~28.4 seconds** for the check-gateway timeout and routing table recalculation to complete.

This can be further improved with scripts and I'm working on it!

**CONVERGENCE TIME:**



| Load Balancing Profile | Connection Integrity | State Path Failure Action | Total Convergence Time |
|------------------------|---------------------|---------------------------|------------------------|
| Standard Static Route | Broken / Session Blackhole | None (Stuck to Local GW) | Infinite Timeout |
| Recursive Routing | Sticky / Re-Routed | Pivots to Healthy WAN | ~28.4 Seconds |

**HOW LONG THE CONVERGENCE IS WHEN LINK IS DOWN:**



Only one packet dropped!

---

## Challenges

**The Asymmetric Return-Path Conflict**

**Challenge:** Oh boy recursive routing may look easy, but when integrated with PCC and backup routes it's a nightmare!.

**How I solved it:** Implemented Policy-Based Routing (PBR) by marking connections as they entered specific WAN interfaces. I then used routing-marks to ensure that any packet belonging to a connection that started on ISP 2 was forced to exit through ISP 2, regardless of the default routing distance.

**Recursive Target Selection**

**Challenge:** Using a single recursive target (e.g., only 8.8.8.8) created a single point of failure; if 8.8.8.8 went down, the ISP was marked "Dead" even if the internet was fine.

**How I solved it:** Configured Dual-Target Recursion. The router monitors both 8.8.8.8 and 1.1.1.1 via separate virtual routes. This ensures that the ISP path is only marked inactive if both global DNS providers become unreachable, preventing false-positive failover events.

---

## Final Thoughts

This project demonstrates the necessity of path-aware load balancing in modern networks.

By combining PCC's granular hashing with the "intelligent" health checks of recursive routing, I've built a network that not only doubles available bandwidth but also self-heals during complex ISP outages that would leave standard configurations like ECMP,PFIFO,etc. paralyzed.



---

## The Proposal

![Network Topology](https://YOUR-IMAGE-HOST.com/topology.png)

**Traffic Distribution Engine:** Per-Connection Classifier (PCC) using both-addresses-and-ports hashing for granular flow distribution

**Failover Mechanism:** Recursive Route Lookups utilizing global DNS (8.8.8.8 / 1.1.1.1) as scope targets to verify true end-to-end internet reachability beyond the immediate ISP gateway

**Path Engineering:** Policy-Based Routing (PBR) via IP Mangle rules to enforce path affinity and avoid asymmetric routing breaks

*Second project down. More to come.* 🔥
