# Dual-WAN Enterprise Gateway: PCC Load Balancing & Recursive Routing Failover
### *My Ninth Project as a Network Engineer*

> Don't trust an ISP just because the cable is plugged in.

---

## 🗺️ The Topology

![Network Topology](https://YOUR-IMAGE-HOST.com/topology.png)

**Traffic Distribution Engine:** Per-Connection Classifier (PCC) using both-addresses-and-ports hashing for granular flow distribution

**Failover Mechanism:** Recursive Route Lookups utilizing global DNS (8.8.8.8 / 1.1.1.1) as scope targets to verify true end-to-end internet reachability beyond the immediate ISP gateway

**Path Engineering:** Policy-Based Routing (PBR) via IP Mangle rules to enforce path affinity and avoid asymmetric routing breaks

---

## 🎭 The Lore

Okay so here's the scenario:

Standard Dual-WAN setups often suffer from two critical flaws: Asymmetric Session Breaks and "Ghost" Gateway Failures.

If a router uses simple Round-Robin packet distribution, secure websites (HTTPS/Banking) will drop the session because the source IP appears to flip-flop every second.

Furthermore, if the ISP's physical modem stays "Up" but their internal network loses internet connectivity, a standard router remains "blind" to the failure and continues sending packets into a black hole.

I engineered a solution that ensures both High-Speed Aggregation and Intelligent Self-Healing:

1. **PCC Hashing & Stickiness:** I utilized the PCC matcher to hash traffic based on source/destination IPs and ports. This ensures that once a session is established on ISP 1, it "sticks" there, preserving stateful connections while allowing different clients (or different browser tabs) to utilize the full capacity of both ISPs.

2. **Recursive Monitoring:** I decoupled the route "Check-Gateway" from the ISP's local IP. By routing toward a virtual target (like Google DNS) via the specific ISP gateway, the router only considers the path "Active" if it can reach the actual internet. If the path fails, the routing table dynamically pivots all traffic to the healthy ISP within seconds.

---

## 🛠️ Performance Highlights

**Granular Multi-WAN Hashing**  
Deployed a complete PCC engine capable of tracking separate network streams and dynamically sharing the transport load based on client transport definitions.

**Failover Verification**  
Constructed a multi-hop recursive routing path that targets global web nodes to prevent gateway blackholing during upstream carrier outages.

**Session-State Enforcement**  
Anchored connection tracking boundaries across the Mangle table to prevent multi-WAN packet reordering and keep secure user connections stable.

![PCC Load Balancing Demo](https://github.com/user-attachments/assets/0a27f1f6-234f-4fe4-9e7e-617283c09e23)

---

## 🧪 The Proof

### Test 1: Multi-Client PCC Hashing Verification

**What I did:** Monitored the IP Mangle table and Connection Tracker while initiating concurrent traffic from two distinct Kali Linux clients running on VLAN 10 and VLAN 20.

**What happened:** The PCC hashing engine successfully identified the unique source/destination pairs. Connection Marks inside the WinBox tracking suite confirmed that the VLAN 10 client was hashed to ISP 1, while the VLAN 20 client was concurrently hashed to ISP 2, proving successful load distribution across the multi-WAN fabric.

![PCC Hashing](https://YOUR-IMAGE-HOST.com/pcc-hashing.gif)

---

### Test 2: Public IP Hashing & Session Stickiness

**What I did:** Opened multiple independent browser sessions on a single host to query Public IP discovery websites and repeatedly refreshed the pages to verify session persistence.

**What happened:** Different browser tabs successfully hashed to different ISPs (Tab A showed ISP 1's public IP, Tab B showed ISP 2's). Despite multiple heavy refreshes, each specific session remained pinned to its respective ISP, confirming that PCC maintains stateful affinity and prevents session-reset issues.

![Session Stickiness](https://YOUR-IMAGE-HOST.com/session-stickiness.gif)

---

### Test 3: ECMP vs. PCC - Debunking the Round-Robin Myth

**What I did:** Switched the core routing configuration to Equal-Cost Multi-Path (ECMP) to analyze the differences in traffic hashing under stress compared to PCC.

**What happened:** Testing successfully debunked a common networking myth. Many assume ECMP sends packets via simple, alternating round-robin distribution which breaks active sessions on a refresh. In reality, once an ECMP connection is established, it hashes and sticks to that single ISP. When the webpage was refreshed, the session remained completely sticky on its path, proving ECMP keeps connection stickiness intact but lacks the granular control over specific traffic profiles that PCC provides through Mangle rules.

![ECMP vs PCC](https://YOUR-IMAGE-HOST.com/ecmp-vs-pcc.png)

---

### Test 4: Recursive Route Failover Dynamics & Convergence Timing

**What I did:** Simulated a "Ghost" gateway failure by creating an explicit packet drop rule for the recursive targets (8.8.8.8 and 1.1.1.1) on ISP 1 while keeping the physical gateway interface active, using a high-frequency ping (0.1s interval) to measure the exact failover time.

**What happened:** Without recursive routing, the connections became confused and packets were dropped into an active black hole because the router thought the up-state interface was healthy.

With recursive routing active, the router detected the loss of 8.8.8.8/1.1.1.1 reachability. Traceroute logs showed the path initially exiting via ISP 1, then dynamically shifting to ISP 2 the moment the recursive check failed. The high-frequency ping recorded 340 dropped packets and 56 received during the transition. This calculated to a total failover convergence time of exactly **~28.4 seconds** for the check-gateway timeout and routing table recalculation to complete.

| Load Balancing Profile | Connection Integrity | State Path Failure Action | Total Convergence Time |
|------------------------|---------------------|---------------------------|------------------------|
| Standard Static Route | Broken / Session Blackhole | None (Stuck to Local GW) | Infinite Timeout |
| Recursive Routing | Sticky / Re-Routed | Pivots to Healthy WAN | ~28.4 Seconds |

![Recursive Failover](https://YOUR-IMAGE-HOST.com/recursive-failover.gif)

---

## 🚧 Engineering Challenges

**The Asymmetric Return-Path Conflict**

**Challenge:** Packets entering from ISP 2 were being responded to via the Default Route (ISP 1), causing the ISP to drop the "unsolicited" traffic.

**How I solved it:** Implemented Policy-Based Routing (PBR) by marking connections as they entered specific WAN interfaces. I then used routing-marks to ensure that any packet belonging to a connection that started on ISP 2 was forced to exit through ISP 2, regardless of the default routing distance.

**Recursive Target Selection**

**Challenge:** Using a single recursive target (e.g., only 8.8.8.8) created a single point of failure; if 8.8.8.8 went down, the ISP was marked "Dead" even if the internet was fine.

**How I solved it:** Configured Dual-Target Recursion. The router monitors both 8.8.8.8 and 1.1.1.1 via separate virtual routes. This ensures that the ISP path is only marked inactive if both global DNS providers become unreachable, preventing false-positive failover events.

---

## 📊 Summary

| Component | Configuration | Why |
|-----------|---------------|-----|
| **PCC Hashing** | both-addresses-and-ports | Session stickiness across multiple ISPs |
| **Recursive Routing** | 8.8.8.8 + 1.1.1.1 monitoring | Prevents "ghost" gateway failures |
| **PBR Mangle Rules** | Connection marking per WAN | Enforces symmetric return path |
| **Failover Time** | ~28.4 seconds | Self-healing without manual intervention |

---

## 💡 Final Thoughts

This project demonstrates the necessity of path-aware load balancing in modern networks.

By combining PCC's granular hashing with the "intelligent" health checks of recursive routing, I've built a network that not only doubles available bandwidth but also self-heals during complex ISP outages that would leave standard configurations paralyzed.

**The Metric That Matters: Failover Convergence Time**

I measured this by simulating a "ghost" outage while running high-frequency pings. Standard static routes would have failed indefinitely. Recursive routing detected the upstream loss and pivoted traffic within ~28.4 seconds.

**Why This Matters to an Employer:**

Most engineers set up Dual-WAN with basic failover. They test by unplugging the cable. That's not good enough.

I test the hard way. The ISP modem is up. The link light is green. But the internet is dead. My router knows the difference.

28 seconds of disruption might be acceptable. Or it might not. The point is: **I know the number.** And I know how to tune it.

That's the difference between "it works" and "it's engineered."

---

*Ninth project down. More to come.* 🔥
