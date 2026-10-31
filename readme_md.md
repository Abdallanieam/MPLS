# 🌐 MPLS Network Infrastructure Simulation & Verification

![GNS3](https://img.shields.io/badge/Simulator-GNS3-blue.svg)
![Routing Protocol](https://img.shields.io/badge/IGP-OSPF-orange.svg)
![Label Switching](https://img.shields.io/badge/Core-MPLS%20%2F%20LDP-green.svg)
![Wireshark](https://img.shields.io/badge/Analysis-Wireshark-167DA5.svg)

A comprehensive lab implementation and traffic analysis of a **Multi-Protocol Label Switching (MPLS)** core infrastructure using **GNS3** and **Cisco IOS Routers**.

This project demonstrates end-to-end IP reachability, interior gateway routing via **OSPF**, label distribution via **LDP**, dynamic label switching verification (`traceroute`), and packet-level inspection using **Wireshark**.

---

## 📌 Network Topology Overview

The network model consists of a service provider (ISP) core backbone connecting customer edge devices:

```text
[CE1] <---> (Ingress PE1) <---> [ P1 ] <---> [ P2 ] <---> (Egress PE2) <---> [CE2]
```

### 🔹 Node Roles & IP Addressing Scheme

| Node | Role / Description | Interface | IP Address | Subnet Mask |
| :--- | :--- | :--- | :--- | :--- |
| **CE1** | Customer Edge (Sender) | `f0/0`<br>`Loopback1` | `10.10.10.1/24`<br>`1.1.1.1/32` | `255.255.255.0`<br>`255.255.255.255` |
| **PE1** | Provider Edge (Ingress LDP) | `f0/0` (to CE1)<br>`f0/1` (to P1) | `10.10.10.2/24`<br>`20.20.20.1/24` | `255.255.255.0`<br>`255.255.255.0` |
| **P1** | Provider Core Router | `f0/1` (to PE1)<br>`f0/0` (to P2) | `20.20.20.2/24`<br>`30.30.30.1/24` | `255.255.255.0`<br>`255.255.255.0` |
| **P2** | Provider Core Router | `f0/0` (to P1)<br>`f0/1` (to PE2) | `30.30.30.2/24`<br>`40.40.40.1/24` | `255.255.255.0`<br>`255.255.255.0` |
| **PE2** | Provider Edge (Egress LDP) | `f0/1` (to P2)<br>`f0/0` (to CE2) | `40.40.40.2/24`<br>`50.50.50.1/24` | `255.255.255.0`<br>`255.255.255.0` |
| **CE2** | Customer Edge (Receiver) | `f0/0`<br>`Loopback1` | `50.50.50.2/24`<br>`2.2.2.2/32` | `255.255.255.0`<br>`255.255.255.255` |

---

## 🛠️ Key Technical Highlights

1. **OSPF Routing Protocol (Area 0):**
   - Configured across core ISP routers (`PE1`, `P1`, `P2`, `PE2`) for full internal reachability.
   - Passive interfaces applied towards CE routers (`CE1` and `CE2`) for security and optimization.
2. **MPLS & LDP (Label Distribution Protocol):**
   - Enabled globally (`mpls ip`) and per-interface across the ISP backbone.
   - Dynamic label binding across core nodes (`Label 100-199` on PE1, `Label 200-299` on P1, `Label 300-399` on P2, `Label 400-499` on PE2).
3. **Verification & Packet Analysis:**
   - Multi-hop label tracking verified via Cisco `traceroute`.
   - Wireshark captures confirming MPLS shim header encapsulation (`Label: 200`, `Exp: 0`, `S: 1`, `TTL: 253`).

---

## 🚀 Setup & Configuration Steps

### 1️⃣ Basic Interface & IP Configuration
Enable interfaces and assign IP addresses according to the topology table. Example for **PE1**:
```cisco
PE1(config)# interface FastEthernet 0/0
PE1(config-if)# ip address 10.10.10.2 255.255.255.0
PE1(config-if)# no shutdown

PE1(config)# interface FastEthernet 0/1
PE1(config-if)# ip address 20.20.20.1 255.255.255.0
PE1(config-if)# no shutdown
```

### 2️⃣ OSPF Routing Configuration
Configure OSPF on Provider routers and set customer-facing interfaces to passive:
```cisco
PE1(config)# router ospf 1
PE1(config-router)# network 0.0.0.0 255.255.255.255 area 0
PE1(config-router)# passive-interface FastEthernet 0/0
```

### 3️⃣ MPLS & LDP Enablement
Activate MPLS IP switching globally and on core-facing interfaces:
```cisco
PE1(config)# mpls ip
PE1(config)# interface FastEthernet 0/1
PE1(config-if)# mpls ip
```
*(Repeat same core interface setup on P1, P2, and PE2)*

---

## 🔍 Verification & Testing

### 🟢 1. Ping Verification (End-to-End Connectivity)
Testing connectivity from **CE1** to destination IP **50.50.50.2** (CE2 interface):
```cisco
CE1# ping 50.50.50.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 50.50.50.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 132/146/156 ms
```

### 🏷️ 2. Traceroute Verification (Label Switching Path)
Executing a `traceroute` from **CE1** shows MPLS label headers inserted along the ISP core:
```cisco
CE1# traceroute 50.50.50.2
Type escape sequence to abort.
Tracing the route to 50.50.50.2
VRF info: (vrf in name/id, vrf out name/id)
  1 10.10.10.2 16 msec 32 msec 20 msec
  2 20.20.20.2 [MPLS: Label 202 Exp 0] 128 msec 116 msec 120 msec
  3 30.30.30.2 [MPLS: Label 302 Exp 0] 68 msec 72 msec 92 msec
  4 40.40.40.2 124 msec 116 msec 120 msec
  5 50.50.50.2 156 msec 144 msec 164 msec
```

### 📊 3. Checking MPLS Forwarding Table (LIB / LFIB)
Checking the Label Forwarding Information Base on **PE1**:
```cisco
PE1# show mpls forwarding-table
Local   Outgoing    Prefix           Bytes Label   Outgoing     Next Hop
Label   Label       or Tunnel Id     Switched      Interface
100     Pop Label   30.30.30.1/24    0             Fa0/1        20.20.20.2
101     201         40.40.40.0/24    0             Fa0/1        20.20.20.2
102     202         50.50.50.0/24    0             Fa0/1        20.20.20.2
```

---

## 🦈 Wireshark Packet Capture Analysis

Packet capture taken on the link between **P1** and **P2** during ICMP traffic:

```text
+-------------------------------------------------------------------+
| Ethernet II, Src: ca:03:56:b8:00:08, Dst: ca:02:32:84:00:08       |
+-------------------------------------------------------------------+
| MultiProtocol Label Switching Header                              |
|   MPLS Label: 200                                                 |
|   Experimental Bits (Exp): 0                                      |
|   Bottom of Label Stack (S): 1                                    |
|   Time to Live (TTL): 253                                         |
+-------------------------------------------------------------------+
| Internet Protocol Version 4, Src: 50.50.50.2, Dst: 10.10.10.1    |
+-------------------------------------------------------------------+
| Internet Control Message Protocol (ICMP Echo Reply)               |
+-------------------------------------------------------------------+
```

---

## 📂 Project Structure

```text
.
├── topology/
│   └── mpls_gns3_topology.net   # GNS3 Topology File
├── configs/
│   ├── CE1.cfg                  # Cisco IOS Config for CE1
│   ├── PE1.cfg                  # Cisco IOS Config for PE1
│   ├── P1.cfg                   # Cisco IOS Config for P1
│   ├── P2.cfg                   # Cisco IOS Config for P2
│   ├── PE2.cfg                  # Cisco IOS Config for PE2
│   └── CE2.cfg                  # Cisco IOS Config for CE2
├── captures/
│   └── mpls_icmp_capture.pcapng # Wireshark Packet Capture File
└── README.md                    # Project Documentation
```

---

## 🤝 Contributing & License
Distributed under the MIT License. Feel free to fork, update, and submit pull requests!