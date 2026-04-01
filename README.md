#  Lab 2 — TCP/IP Network Attacks: SYN Flood, ARP Cache Poisoning & ICMP Redirect

**Subject:** 48730/32548 Cybersecurity — University of Technology Sydney  
**Tools Used:** Netwag, Wireshark, tshark, arp  
**Skills Demonstrated:** DoS attack simulation, Layer 2 spoofing, routing manipulation, network traffic analysis

---

## Overview

This lab explores three foundational network-layer attacks that exploit weaknesses in core TCP/IP protocols. These are not application-level vulnerabilities — they target the protocols themselves: TCP's three-way handshake, ARP's trust-based design, and ICMP's redirect mechanism. Understanding how these attacks work at a packet level is essential for both network defence and incident response.

The lab environment consisted of three VMs: an attacker machine, a client, and a server — all on an isolated internal network.

---

## Task 1 — SYN Flood Attack (Denial of Service)

### The vulnerability being exploited
TCP connections are established through a three-way handshake: the client sends SYN, the server replies SYN-ACK and holds the connection in a half-open state while waiting for the final ACK. The server has a finite queue for these half-open connections.

A SYN flood exploits this by sending enormous volumes of SYN packets with spoofed source IP addresses. The server replies to fake IPs that never complete the handshake, filling its connection queue until it can no longer accept legitimate connections. This is a classic **Denial of Service (DoS)** attack.

### What I did
Configured Netwag Tool 76 (SYN flood) on the attacker VM, targeting the client at `10.0.2.8` on port 80 with best-effort IP spoofing enabled. Monitored the resulting traffic on the client using `tshark`.

### What I observed
The tshark output showed a continuous flood of TCP SYN packets arriving from hundreds of different spoofed source IPs, all targeting port 80. The client's TCP stack was being forced to allocate resources for each half-open connection. In a production environment this would exhaust the connection table and cause service disruption for legitimate users.

### Severity assessment
This is a **high severity** DoS attack. It does not require authentication or exploitation of an application bug — just network access and the ability to send spoofed packets. SYN floods are one of the most common attack types seen in the wild, including in large-scale DDoS campaigns.

### Mitigations
- **SYN cookies:** Instead of allocating state immediately, the server encodes connection info in the SYN-ACK sequence number. State is only allocated when the final ACK is received with a valid cookie, eliminating the half-open queue problem.
- **Rate limiting** on inbound SYN packets at the firewall or ISP level.
- **Upstream DDoS scrubbing** services for production environments.

---

## Task 2 — ARP Cache Poisoning (Layer 2 Spoofing)

### The vulnerability being exploited
ARP (Address Resolution Protocol) maps IP addresses to MAC addresses on a local network. The critical weakness: ARP has no authentication mechanism. Any device can send an unsolicited ARP reply claiming to be the owner of any IP address, and most operating systems will accept and cache it without verification.

An attacker can exploit this by continuously broadcasting fake ARP replies, poisoning the ARP cache of target machines so that traffic intended for one IP gets sent to the attacker's MAC address instead. This enables **man-in-the-middle (MitM)** attacks or traffic redirection.

### What I did
First recorded the legitimate ARP table on the server using `arp -a`, confirming the real MAC address for each IP. Then configured Netwag Tool 80 (Periodically Send ARP Replies) on the attacker VM, sending spoofed ARP replies that mapped a legitimate IP (`10.0.2.8`) to a fake MAC address (`00:50:56:38:05:00`). After running the tool, re-checked the ARP table on the server.

### What I observed
The ARP cache on the server updated automatically. The entry for `10.0.2.8` now pointed to the attacker's fake MAC address instead of the legitimate one. Traffic from the server destined for `10.0.2.8` would now be forwarded to the wrong host — without any indication to the server that anything was wrong.

### Why this is dangerous
Once ARP poisoning is successful, the attacker sits between two communicating hosts and can intercept, read, or modify all traffic passing between them — even if the traffic uses application-level authentication. Credentials, session tokens, and sensitive data are all exposed if the application layer does not use end-to-end encryption.

### Mitigation
- **Static ARP entries** for critical systems (routers, servers) prevent dynamic updates from overwriting known-good mappings.
- **Dynamic ARP Inspection (DAI)** on managed switches validates ARP packets against a trusted DHCP snooping binding table, dropping spoofed replies automatically.
- Network segmentation to limit the blast radius of Layer 2 attacks.

---

## Task 3 — ICMP Redirect Attack (Routing Manipulation)

### The vulnerability being exploited
ICMP Redirect messages are a legitimate network mechanism: a router sends an ICMP Type 5 (Redirect) message to a host to tell it a more efficient route exists for a destination. The host updates its routing table accordingly.

The problem: ICMP has no authentication. An attacker on the network can forge ICMP Redirect messages and send them to a victim, tricking the victim's machine into re-routing its traffic through an attacker-controlled gateway. This is another path to a **man-in-the-middle position**.

### What I did
Set up Wireshark on the client VM with an ICMP filter to monitor traffic, then started a continuous ping from the client to the server (`10.0.2.6`) to establish a baseline. On the attacker VM, configured Netwag Tool 86 (Sniff and Send ICMP Redirect) with the attacker's IP (`10.0.2.7`) as the new gateway.

### What I observed
Wireshark on the client showed ICMP Redirect packets arriving with `Type: 5 (Redirect for host)` and the attacker's IP as the new gateway. The client's routing table was updated, and subsequent ICMP echo requests were routed through the attacker's machine. The attacker had successfully inserted themselves into the traffic path without the client's knowledge.

### Why this matters
Unlike ARP poisoning which works at Layer 2, ICMP redirect attacks manipulate the routing table at Layer 3, making them effective across subnets in some configurations. In an unmonitored network, this attack could persist silently for extended periods.

### Mitigation
- **Disable ICMP redirect acceptance** on all endpoints: `net.ipv4.conf.all.accept_redirects = 0` in Linux sysctl settings.
- Use **static routes** for critical paths where possible.
- Deploy **secure routing protocols with authentication** (e.g. OSPFv3 with IPsec, BGP with MD5/TCP-AO) on routers to prevent spoofed routing updates.
- Network monitoring: anomalous ICMP Redirect messages from unexpected sources should trigger alerts.

---

## Comparative Analysis

| Attack | Protocol Exploited | Layer | Impact | Detectable by IDS? |
|---|---|---|---|---|
| SYN Flood | TCP | Layer 4 | Denial of Service | Yes — high SYN rate from many IPs |
| ARP Poisoning | ARP | Layer 2 | MitM / Traffic Redirection | Yes — duplicate IP-to-MAC mappings |
| ICMP Redirect | ICMP | Layer 3 | Routing Manipulation / MitM | Yes — unexpected ICMP Type 5 messages |

---

## Key Takeaways

All three of these attacks succeed because the underlying protocols were designed in an era when the internet was a trusted academic network. Authentication was not built in. TCP assumes SYN packets are genuine. ARP assumes replies are legitimate. ICMP assumes redirect messages come from valid routers.

Modern defences address this through compensating controls — SYN cookies, DAI, sysctl hardening — rather than changes to the protocols themselves. Understanding the root cause of each vulnerability is what allows a security professional to implement the right control rather than just following a checklist.
