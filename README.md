# Junos-Inter-AS-Option-C-Option-B-on-the-Same-PE

This lab validates a hybrid Inter-AS design where a single Junos PE terminates Option C (BGP Labeled-Unicast) on one side and Option B (VPNv4) on the other.

<img width="1367" height="472" alt="image" src="https://github.com/user-attachments/assets/1b0005ba-6705-46f8-9970-de72d3a13690" />

---

This lab validates a **hybrid Inter-AS design** where a single Junos PE terminates **Option C (BGP Labeled-Unicast)** on one side and **Option B (VPNv4)** on the other. The goal is to support **Carrier-of-Carrier VPNs** one carrier transporting another carrier’s L3VPN routes across multiple AS domains **with scale, flexibility, and operational clarity**.

**Device Under Test (DUT):** `vMX2 (AS65002)`
* **Upstream:** `vMX1 (AS65001)` via **Option C (LU)**
* **Downstream:** `vMX3 (AS65003)` via **Option B (VPNv4)**

---

## Introduction & Purpose

Many operators need to interconnect **multiple autonomous systems** while preserving **MPLS VPN context** and scaling cleanly. This design shows how Junos can **terminate Option C on one side and Option B on the other within the same PE**, enabling:

* **Carrier-of-Carrier VPNs** (one carrier transporting another carrier’s L3VPNs)
* **Incremental migrations** (e.g., LU-based peering ↔ VPNv4-based peering)
* **Operational flexibility** (policy-driven leaking, selective RT import/export, explicit label control)

The validation covers both **control-plane** (labels, RTs, RDs, AS paths) and **data-plane** (MPLS imposition/swap, ICMP, MPLS-aware traceroute).

---

## Flow Summary

1. **Prefix origination (vP1 → vMX1, AS65001)**

   * Customer prefix `172.16.10.1/32` (and `.10.2/32`) originates in VRF **RED** via OSPF.
   * **vMX1** leaks these routes into **BGP Labeled-Unicast (LU)** for inter-AS **Option C**.

2. **Option C hop (vMX1 ↔ vMX2)**

   * vMX1 exports the OSPF-learned /32s into LU and advertises them to **vMX2**.
   * **Verified:** vMX2 receives both prefixes with **LU label `300112`** and AS path `[65001 I]`.

3. **Option B hop (vMX2 ↔ vMX3)**

   * vMX2 re-advertises these VRF routes as **VPNv4** with **RT `target:65000:100`**.
   * vMX2 allocates **VPN labels `300576` and `300592`** for the two /32s.
   * **vMX3** imports the routes into VRF RED and installs them with appropriate MPLS labels.

4. **Redistribution to CE (vMX3 → vP3, AS65003)**

   * vMX3 redistributes VPNv4 routes into **OSPF as E2** routes.
   * CE **vP3** learns `172.16.10.1/32` and `172.16.10.2/32` as **OSPF externals**.

5. **Data-plane validation**

   * Successful **ping** and **traceroute** from **vP1** (source `172.16.10.1`) to **vP3** (`172.16.30.1`), showing **MPLS labels** being pushed and swapped correctly across the **Option C** and **Option B** legs.

---

## CLI Evidence (as-built)

> These are the exact snippets captured during the validation to aid replication and troubleshooting.

### vP1 — `show route` + dataplane

```
root@vP1> show route table RED.inet.0 

RED.inet.0: 8 destinations, 9 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

172.16.10.1/32     *[Direct/0] 1w0d 01:52:53
                    >  via lo0.100
172.16.10.2/32     *[Direct/0] 03:02:52
                    >  via ge-0/0/2.0
                    [Local/0] 03:02:52
                       Local via ge-0/0/2.0
172.16.30.1/32     *[OSPF/150] 02:59:40, metric 0, tag 0
                    >  to 10.0.1.1 via ge-0/0/1.0
172.16.30.2/32     *[OSPF/150] 02:59:40, metric 0, tag 0
                    >  to 10.0.1.1 via ge-0/0/1.0
224.0.0.5/32       *[OSPF/10] 1w0d 01:52:54, metric 1
                       MultiRecv
```

```
root@vP1> ping 172.16.30.1 routing-instance RED source 172.16.10.1 detail 
PING 172.16.30.1 (172.16.30.1): 56 data bytes
64 bytes from 172.16.30.1 via ge-0/0/1.0: icmp_seq=0 ttl=62 time=5.407 ms
64 bytes from 172.16.30.1 via ge-0/0/1.0: icmp_seq=1 ttl=62 time=5.727 ms
64 bytes from 172.16.30.1 via ge-0/0/1.0: icmp_seq=2 ttl=62 time=5.519 ms
64 bytes from 172.16.30.1 via ge-0/0/1.0: icmp_seq=3 ttl=62 time=4.936 ms
64 bytes from 172.16.30.1 via ge-0/0/1.0: icmp_seq=4 ttl=62 time=5.678 ms
^C
--- 172.16.30.1 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 4.936/5.453/5.727/0.283 ms
```

```
root@vP1> traceroute 172.16.30.1 routing-instance RED source 172.16.10.1 
traceroute to 172.16.30.1 (172.16.30.1) from 172.16.10.1, 30 hops max, 52 byte packets
 1  10.0.1.1 (10.0.1.1)  2.661 ms  1.904 ms  2.291 ms
 2  10.0.12.2 (10.0.12.2)  3.870 ms  3.482 ms  3.110 ms
     MPLS Label=300560 CoS=0 TTL=1 S=1
 3  10.0.23.1 (10.0.23.1)  4.308 ms  3.805 ms  4.327 ms
     MPLS Label=300016 CoS=0 TTL=1 S=1
 4  172.16.30.1 (172.16.30.1)  4.002 ms  4.111 ms  4.324 ms
```

### vMX1 — Option C (LU) toward vMX2

```
root@VMX1_re> show bgp summary 
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0               
                       0          0          0          0          0          0
inet.3               
                       0          0          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.12.2             65002        446        447       0       2     3:19:31 Establ
  RED.inet.0: 3/3/3/0
```

```
root@VMX1_re> show route advertising-protocol bgp 10.0.12.2 detail 

RED.inet.0: 10 destinations, 10 routes (10 active, 0 holddown, 0 hidden)
* 172.16.10.1/32 (1 entry, 1 announced)
 BGP group TO-VMX2 type External
     Route Label: 300112  <<<<
     Nexthop: Self
     Flags: Nexthop Change
     MED: 0
     AS path: [65001] I 
     Communities: rte-type:0.0.0.0:5:1
     Entropy label capable

* 172.16.10.2/32 (1 entry, 1 announced)
 BGP group TO-VMX2 type External
     Route Label: 300112 <<<<
     Nexthop: Self
     Flags: Nexthop Change
     MED: 0
     AS path: [65001] I 
     Communities: rte-type:0.0.0.0:5:1
     Entropy label capable
```

### vMX2 (DUT) — Receives LU, re-advertises VPNv4

```
root@VMX2_re> show bgp summary 
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 2 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0               
                       0          0          0          0          0          0
bgp.l3vpn.0          
                       3          3          0          0          0          0
inet.3               
                       0          0          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.12.1             65001        456        453       0       3     3:23:12 Establ
  RED.inet.0: 2/2/2/0
10.0.23.1             65003      13484      13433       0       2  4d 5:05:14 Establ
  bgp.l3vpn.0: 3/3/3/0
  RED.inet.0: 3/3/3/0  
```

```
root@VMX2_re> show route receive-protocol bgp 10.0.12.1 detail 

inet.0: 10 destinations, 10 routes (10 active, 0 holddown, 0 hidden)

RED.inet.0: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
* 172.16.10.1/32 (1 entry, 1 announced)
     Accepted
     Route Label: 300112 <<<<
     Nexthop: 10.0.12.1
     MED: 0
     AS path: 65001 I 
     Communities: rte-type:0.0.0.0:5:1
     Entropy label capable, next hop field matches route next hop

* 172.16.10.2/32 (1 entry, 1 announced)
     Accepted
     Route Label: 300112 <<<<
     Nexthop: 10.0.12.1
     MED: 0
     AS path: 65001 I 
     Communities: rte-type:0.0.0.0:5:1
     Entropy label capable, next hop field matches route next hop
```

```
root@VMX2_re> show route table bgp.l3vpn.0 

bgp.l3vpn.0: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

65002:100:10.0.12.0/30                
                   *[Direct/0] 4d 05:07:33
                    >  via ge-0/0/1.0
65002:100:10.0.12.2/32                
                   *[Local/0] 4d 05:07:33
                       Local via ge-0/0/1.0
65002:100:172.16.10.1/32                
                   *[BGP/170] 03:25:23, MED 0, localpref 100
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.0.12.1 via ge-0/0/1.0, Push 300112
65002:100:172.16.10.2/32                
                   *[BGP/170] 03:22:14, MED 0, localpref 100
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.0.12.1 via ge-0/0/1.0, Push 300112
65003:100:10.0.3.0/30                
                   *[BGP/170] 4d 05:06:54, localpref 100
                      AS path: 65003 I, validation-state: unverified
                    >  to 10.0.23.1 via ge-0/0/2.0, Push 300032
65003:100:172.16.30.1/32                
                   *[BGP/170] 4d 05:07:25, localpref 100
                      AS path: 65003 I, validation-state: unverified
                    >  to 10.0.23.1 via ge-0/0/2.0, Push 300016
65003:100:172.16.30.2/32                
                   *[BGP/170] 4d 05:06:53, MED 1, localpref 100
                      AS path: 65003 I, validation-state: unverified
                    >  to 10.0.23.1 via ge-0/0/2.0, Push 300032
```

### vMX3 — Imports VPNv4, redistributes to OSPF

```
root@VMX3_re> show bgp summary 
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.l3vpn.0          
                       2          2          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.23.2             65002      13440      13489       0       2  4d 5:08:21 Establ
  bgp.l3vpn.0: 2/2/2/0
  RED.inet.0: 2/2/2/0
```

```
root@VMX3_re> show route receive-protocol bgp 10.0.23.2 detail 

inet.0: 9 destinations, 9 routes (9 active, 0 holddown, 0 hidden)

RED.inet.0: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
* 172.16.10.1/32 (1 entry, 1 announced)
     Import Accepted
     Route Distinguisher: 65002:100
     VPN Label: 300576
     Nexthop: 10.0.23.2
     AS path: 65002 65001 I 
     Communities: target:65000:100 rte-type:0.0.0.0:5:1

* 172.16.10.2/32 (1 entry, 1 announced)
     Import Accepted
     Route Distinguisher: 65002:100
     VPN Label: 300592
     Nexthop: 10.0.23.2
     AS path: 65002 65001 I 
     Communities: target:65000:100 rte-type:0.0.0.0:5:1

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)

mpls.0: 6 destinations, 6 routes (6 active, 0 holddown, 0 hidden)

bgp.l3vpn.0: 6 destinations, 6 routes (6 active, 0 holddown, 0 hidden)

* 65002:100:172.16.10.1/32 (1 entry, 0 announced)
     Import Accepted
     Route Distinguisher: 65002:100
     VPN Label: 300576
     Nexthop: 10.0.23.2
     AS path: 65002 65001 I 
     Communities: target:65000:100 rte-type:0.0.0.0:5:1

* 65002:100:172.16.10.2/32 (1 entry, 0 announced)
     Import Accepted
     Route Distinguisher: 65002:100
     VPN Label: 300592
     Nexthop: 10.0.23.2
     AS path: 65002 65001 I 
     Communities: target:65000:100 rte-type:0.0.0.0:5:1

inet6.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)

RED.inet6.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
```

```
root@VMX3_re> show route table RED.inet.0 

RED.inet.0: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

172.16.10.1/32     *[BGP/170] 04:13:29, localpref 100
                      AS path: 65002 65001 I, validation-state: unverified
                    >  to 10.0.23.2 via ge-0/0/2.0, Push 300576
172.16.10.2/32     *[BGP/170] 04:10:21, localpref 100
                      AS path: 65002 65001 I, validation-state: unverified
                    >  to 10.0.23.2 via ge-0/0/2.0, Push 300592
172.16.30.1/32     *[Direct/0] 5d 05:12:50
                    >  via lo0.100
172.16.30.2/32     *[OSPF/10] 4d 05:55:02, metric 1
                    >  to 10.0.3.2 via ge-0/0/1.0
224.0.0.5/32       *[OSPF/10] 5d 05:12:51, metric 1
                       MultiRecv
```

### vP3 — OSPF external routes learned

```
root@vP3> show route table RED-vr.inet.0 

RED-vr.inet.0: 6 destinations, 6 routes (6 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.3.0/30        *[Direct/0] 4d 05:56:26
                    >  via ge-0/0/2.0
10.0.3.2/32        *[Local/0] 4d 05:56:26
                       Local via ge-0/0/2.0
172.16.10.1/32     *[OSPF/150] 04:14:08, metric 0, tag 3489725931
                    >  to 10.0.3.1 via ge-0/0/2.0
172.16.10.2/32     *[OSPF/150] 04:11:00, metric 0, tag 3489725931
                    >  to 10.0.3.1 via ge-0/0/2.0
172.16.30.2/32     *[Direct/0] 5d 05:07:33
                    >  via lo0.100
224.0.0.5/32       *[OSPF/10] 5d 05:07:33, metric 1
                       MultiRecv
```

```
root@vP3> show route protocol ospf detail 

inet.0: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)

RED-vr.inet.0: 6 destinations, 6 routes (6 active, 0 holddown, 0 hidden)
172.16.10.1/32 (1 entry, 1 announced)
        *OSPF   Preference: 150
                Next hop type: Router, Next hop index: 588
                Address: 0x70fec1c
                Next-hop reference count: 4
                Next hop: 10.0.3.1 via ge-0/0/2.0, selected
                Session Id: 0x140
                State: <Active Int Ext>
                Age: 4:14:43    Metric: 0 
                Validation State: unverified 
                        Tag: 3489725931 
                Task: RED-vr-OSPF
                Announcement bits (1): 2-KRT 
                AS path: I 
                Thread: junos-main 

172.16.10.2/32 (1 entry, 1 announced)
        *OSPF   Preference: 150
                Next hop type: Router, Next hop index: 588
                Address: 0x70fec1c
                Next-hop reference count: 4
                Next hop: 10.0.3.1 via ge-0/0/2.0, selected
                Session Id: 0x140
                State: <Active Int Ext>
                Age: 4:11:35    Metric: 0 
                Validation State: unverified 
                        Tag: 3489725931 
                Task: RED-vr-OSPF
                Announcement bits (1): 2-KRT 
                AS path: I 
                Thread: junos-main 

224.0.0.5/32 (1 entry, 1 announced)
        *OSPF   Preference: 10
                Next hop type: MultiRecv, Next hop index: 0
                Address: 0x70fe1f4
                Next-hop reference count: 6
                State: <Active NoReadvrt Int>
                Age: 5d 5:08:08         Metric: 1 
                Validation State: unverified 
                Task: OSPF I/O./var/run/ppmd_control
                Announcement bits (1): 2-KRT 
                AS path: I 
                Thread: junos-main 

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)

inet6.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)

RED-vr.inet6.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)

```

```
root@vP3> ping 172.16.10.1 routing-instance RED-vr source 172.16.30.2 detail    
PING 172.16.10.1 (172.16.10.1): 56 data bytes
64 bytes from 172.16.10.1 via ge-0/0/2.0: icmp_seq=0 ttl=61 time=5.296 ms
...
--- 172.16.10.1 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
```

```
root@vP3> traceroute 172.16.10.1 routing-instance RED-vr source 172.16.30.2     
traceroute to 172.16.10.1 (172.16.10.1) from 172.16.30.2, 30 hops max, 52 byte packets
 1  10.0.3.1 (10.0.3.1)  2.511 ms  1.795 ms  1.978 ms
 2  * * *
 3  10.0.12.1 (10.0.12.1)  7.418 ms  4.202 ms  4.032 ms
     MPLS Label=300112 CoS=0 TTL=1 S=1
 4  172.16.10.1 (172.16.10.1)  5.145 ms  5.408 ms  5.375 ms
```

### Lab Configuration 

vP1

        root@vP1> show configuration | display set | except "groups global | apply-groups | groups member0" 
        set version 21.2R3-S2.9
        set system host-name vP1
        set system ports console log-out-on-disconnect
        set interfaces ge-0/0/1 unit 0 family inet address 10.0.1.2/30
        set interfaces ge-0/0/2 unit 0 family inet address 172.16.10.2/32
        set interfaces lo0 unit 100 family inet address 172.16.10.1/32
        set policy-options policy-statement EXP term LOCAL from interface lo0.100
        set policy-options policy-statement EXP term LOCAL from interface ge-0/0/2.0
        set policy-options policy-statement EXP term LOCAL then accept
        set routing-instances RED instance-type vrf
        set routing-instances RED protocols ospf area 0.0.0.0 interface ge-0/0/1.0
        set routing-instances RED protocols ospf export EXP
        set routing-instances RED interface ge-0/0/1.0
        set routing-instances RED interface ge-0/0/2.0
        set routing-instances RED interface lo0.100
        set routing-instances RED route-distinguisher 65001:100
        set routing-instances RED vrf-target target:65000:100
        set routing-options router-id 5.5.5.5
vMX1

        root@VMX1_re> show configuration | display set | except "groups global | apply-groups | groups member0" 
        set version 21.2R3-S2.9
        set system ports console log-out-on-disconnect
        set interfaces ge-0/0/1 unit 0 family inet address 10.0.12.1/30
        set interfaces ge-0/0/1 unit 0 family mpls
        set interfaces ge-0/0/2 unit 0 family inet address 10.0.1.1/30
        set interfaces lo0 unit 0 family inet address 1.1.1.1/32
        set policy-options policy-statement INET0_CE_ROUTE_LEAKING term OSPF from protocol ospf
        set policy-options policy-statement INET0_CE_ROUTE_LEAKING term OSPF from route-filter 172.16.10.1/32 exact
        set policy-options policy-statement INET0_CE_ROUTE_LEAKING term OSPF from route-filter 172.16.10.2/32 exact
        set policy-options policy-statement INET0_CE_ROUTE_LEAKING term OSPF then accept
        set policy-options policy-statement INET0_CE_ROUTE_LEAKING term LAST then reject
        set policy-options policy-statement RED-BGP-TO-OSPF term t1 from protocol bgp
        set policy-options policy-statement RED-BGP-TO-OSPF term t1 then external type 2
        set policy-options policy-statement RED-BGP-TO-OSPF term t1 then accept
        set policy-options policy-statement RED-TO-GLOBAL term t1 from rib RED.inet.0
        set policy-options policy-statement RED-TO-GLOBAL term t1 then accept
        set policy-options community LU-RED members target:65100:100
        set routing-instances RED instance-type vrf
        set routing-instances RED routing-options autonomous-system 65001
        set routing-instances RED protocols bgp group TO-VMX2 type external
        set routing-instances RED protocols bgp group TO-VMX2 neighbor 10.0.12.2 family inet labeled-unicast
        set routing-instances RED protocols bgp group TO-VMX2 neighbor 10.0.12.2 export INET0_CE_ROUTE_LEAKING
        set routing-instances RED protocols bgp group TO-VMX2 neighbor 10.0.12.2 peer-as 65002
        set routing-instances RED protocols ospf area 0.0.0.0 interface ge-0/0/2.0
        set routing-instances RED protocols ospf export RED-BGP-TO-OSPF
        set routing-instances RED interface ge-0/0/1.0
        set routing-instances RED interface ge-0/0/2.0
        set routing-instances RED route-distinguisher 65001:100
        set routing-instances RED vrf-target target:65000:100
        set routing-options router-id 1.1.1.1
        set routing-options autonomous-system 65001
        set protocols mpls interface all

  vMX2 - DUT

        root@VMX2_re> show configuration | display set | except "groups global | apply-groups | groups member0" 
        set version 21.2R3-S2.9
        set system ports console log-out-on-disconnect
        set interfaces ge-0/0/1 unit 0 family inet address 10.0.12.2/30
        set interfaces ge-0/0/1 unit 0 family mpls
        set interfaces ge-0/0/2 unit 0 family inet address 10.0.23.2/30
        set interfaces ge-0/0/2 unit 0 family mpls
        set interfaces lo0 unit 0 family inet address 2.2.2.2/32
        set interfaces lo0 unit 100 family inet
        set policy-options policy-statement EXPORT-RED-BGP term 1 from instance RED
        set policy-options policy-statement EXPORT-RED-BGP term 1 then accept
        set policy-options policy-statement EXPORT-RED-LU-OUT term t1 from rib RED.inet.0
        set policy-options policy-statement EXPORT-RED-LU-OUT term t1 then community add LU-RED
        set policy-options policy-statement EXPORT-RED-LU-OUT term t1 then accept
        set policy-options policy-statement EXPORT-RED-VPNV4 term t1 from instance RED
        set policy-options policy-statement EXPORT-RED-VPNV4 term t1 then accept
        set policy-options policy-statement EXPORT-TO-VMX3 term 1 from instance RED
        set policy-options policy-statement EXPORT-TO-VMX3 term 1 then accept
        set policy-options policy-statement IMPORT-LU-RED term t1 from community LU-RED
        set policy-options policy-statement IMPORT-LU-RED term t1 from community RED-COM
        set policy-options policy-statement IMPORT-LU-RED term t1 then community delete ALL
        set policy-options policy-statement IMPORT-LU-RED term t1 then accept
        set policy-options policy-statement IMPORT-LU-RED then reject
        set policy-options policy-statement MATCH-LU-RED term t1 from community LU-RED
        set policy-options policy-statement MATCH-LU-RED term t1 then accept
        set policy-options policy-statement MATCH-LU-RED then reject
        set policy-options policy-statement RED-TO-GLOBAL term t1 from rib RED.inet.0
        set policy-options policy-statement RED-TO-GLOBAL term t1 then accept
        set policy-options community ALL members *:*
        set policy-options community LU-RED members target:65100:100
        set policy-options community RED-COM members target:65000:100
        set routing-instances RED instance-type vrf
        set routing-instances RED routing-options auto-export
        set routing-instances RED protocols bgp group TO-VMX1-LU type external
        set routing-instances RED protocols bgp group TO-VMX1-LU neighbor 10.0.12.1 family inet labeled-unicast
        set routing-instances RED protocols bgp group TO-VMX1-LU neighbor 10.0.12.1 peer-as 65001
        set routing-instances RED interface ge-0/0/1.0
        set routing-instances RED interface lo0.100
        set routing-instances RED route-distinguisher 65002:100
        set routing-instances RED vrf-import IMPORT-LU-RED
        set routing-instances RED vrf-target target:65000:100
        set routing-options router-id 2.2.2.2
        set routing-options autonomous-system 65002
        set routing-options instance-import RED-TO-GLOBAL
        set routing-options auto-export
        set protocols bgp group TO-VMX3-BGP type external
        set protocols bgp group TO-VMX3-BGP peer-as 65003
        set protocols bgp group TO-VMX3-BGP neighbor 10.0.23.1 family inet-vpn unicast
        set protocols mpls explicit-null
        set protocols mpls label-switched-path LSP-VMX2-to-VMX1 to 1.1.1.1
        set protocols mpls path path1 10.0.12.1 loose
        set protocols mpls interface all

  vMX3

        root@VMX3_re> show configuration | display set | except "groups global | apply-groups | groups member0" 
        set version 21.2R3-S2.9
        set system ports console log-out-on-disconnect
        set interfaces ge-0/0/1 unit 0 family inet address 10.0.3.1/30
        set interfaces ge-0/0/2 unit 0 family inet address 10.0.23.1/30
        set interfaces ge-0/0/2 unit 0 family mpls
        set interfaces lo0 unit 100 family inet address 172.16.30.1/32
        set policy-options policy-statement RED-BGP-TO-OSPF term t1 from protocol bgp
        set policy-options policy-statement RED-BGP-TO-OSPF term t1 then external type 2
        set policy-options policy-statement RED-BGP-TO-OSPF term t1 then accept
        set routing-instances RED instance-type vrf
        set routing-instances RED protocols ospf area 0.0.0.0 interface ge-0/0/1.0
        set routing-instances RED protocols ospf export RED-BGP-TO-OSPF
        set routing-instances RED interface ge-0/0/1.0
        set routing-instances RED interface lo0.100
        set routing-instances RED route-distinguisher 65003:100
        set routing-instances RED vrf-target target:65000:100
        set routing-options router-id 3.3.3.3
        set routing-options autonomous-system 65003
        set protocols bgp group TO-VMX2-BGP type external
        set protocols bgp group TO-VMX2-BGP peer-as 65002
        set protocols bgp group TO-VMX2-BGP neighbor 10.0.23.2 family inet-vpn unicast
        set protocols mpls interface ge-0/0/2.0

vP3

        root@vP3> show configuration | display set | except "groups global | apply-groups | groups member0" 
        set version 21.2R3-S2.9
        set system host-name vP3
        set system ports console log-out-on-disconnect
        set interfaces ge-0/0/2 unit 0 family inet address 10.0.3.2/30
        set interfaces lo0 unit 100 family inet address 172.16.30.2/32
        set routing-instances RED-vr instance-type virtual-router
        set routing-instances RED-vr protocols ospf area 0.0.0.0 interface ge-0/0/2.0
        set routing-instances RED-vr protocols ospf area 0.0.0.0 interface lo0.100 passive
        set routing-instances RED-vr interface ge-0/0/2.0
        set routing-instances RED-vr interface lo0.100
        set routing-options router-id 4.4.4.4   
  
### Final Thoughts     

This lab demonstrates that Junos can seamlessly integrate Inter-AS Option C and Option B on the same PE to deliver true Carrier-of-Carrier L3VPN services, featuring a clean control plane, deterministic labels/RTs, and verified end-to-end forwarding. The approach scales, offers multiple deployment flavours, and keeps operations simple with clear policy hooks for selective leaking and redistribution. In short, it’s a practical, production ready pattern that lets providers interconnect AS domains with confidence while preserving flexibility for future migrations and growth.

📚 References

  * RFC 4364: BGP/MPLS IP Virtual Private Networks (VPNs)
  * RFC 4365: Applicability Statement for BGP/MPLS IP VPNs
  * RFC 3107: Carrying Label Information in BGP-4
