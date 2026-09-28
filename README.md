# 🌐 OSPF Virtual Links — Connecting Discontiguous Areas to the Backbone

![Cisco](https://img.shields.io/badge/Cisco-Packet%20Tracer-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![OSPF](https://img.shields.io/badge/Protocol-OSPF-teal?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
![Level](https://img.shields.io/badge/Level-CCNP-blue?style=for-the-badge)
![Topic](https://img.shields.io/badge/Focus-Virtual%20Links-purple?style=for-the-badge)

> A hands-on Cisco Packet Tracer lab that builds an OSPF design **breaking the "every area must touch Area 0" rule**, watches OSPF complain, and then repairs it with **two chained virtual links** — verified through neighbor tables, virtual-link status, and routing table evidence.

---

## 📖 Overview

OSPF's multi-area design has one non-negotiable rule: every non-backbone area must connect to **Area 0**. When an area can't physically reach the backbone (after a merger, a bad design, or a link that couldn't be run), a **virtual link** can stretch the backbone across an intermediate "transit" area to reach it.

This lab creates exactly that problem, four areas where two of them have no path to Area 0, and then fixes it step by step with virtual links.

**Goals of this lab:**
- Build a topology where Areas 2 and 3 are cut off from the backbone
- Observe how OSPF reports the broken design
- Configure chained virtual links across two transit areas
- Verify virtual-link status, adjacencies, and the resulting routes
- Work out which router pairs can (and cannot) form a virtual link

---

## 🗺️ Network Topology

```
   L1 10.1.1.1              L1 20.1.1.1              L1 30.1.1.1
   (Router ID)              (Router ID)              (Router ID)

 [ AREA 0 ]                [ AREA 1 ]               [ AREA 2 ]            [ AREA 3 ]
 192.168.1.0/24            1.1.1.0/24               2.1.1.0/24            192.168.3.0/24
      │                        │                        │                     │
   ┌──┴───┐   S0/0/0        ┌──┴───┐   g0/1 ─ fg0/1  ┌──┴───┐             LAN of R3
   │  R1  │═══1.1.1.0/24════│  R2  │═══2.1.1.0/24════│  R3  │───[Switch2]──[PCs]
   └──┬───┘                 └──┬───┘                 └──────┘
   [Switch0]                [Switch1]
   [PCs] 192.168.1.x        [PCs] 192.168.2.x  (Area 1)

 R1 = Area 0 + Area 1            R2 = Area 1 + Area 2            R3 = Area 2 + Area 3
 (already touches backbone)      (NO backbone interface)         (NO backbone interface)

 Virtual Link #1:  R1 ◄══ transit Area 1 ══► R2
 Virtual Link #2:  R2 ◄══ transit Area 2 ══► R3
```

| Device | Areas (physical interfaces) | Backbone status                                   |
|--------|--------------------------------|------------------------------------------------------|
| R1     | Area 0, Area 1                  | Attached to the backbone directly                     |
| R2     | Area 1, Area 2                  | **Not attached** — reaches Area 0 through VL #1       |
| R3     | Area 2, Area 3                  | **Not attached** — reaches Area 0 through VL #2 (via R2) |

---

## 🧾 IP Addressing & Area Table

| Device | Interface   | IP Address     | Area | Purpose                    |
|--------|-------------|-----------------|--------|--------------------------------|
| R1     | Gi0/0       | 192.168.1.1     | 0      | LAN1                            |
| R1     | S0/0/0      | 1.1.1.1         | 1      | To R2                           |
| R1     | Loopback1   | 10.1.1.1        | —      | Router ID                       |
| R2     | S0/0/0      | 1.1.1.2         | 1      | To R1                           |
| R2     | Gi0/0       | 192.168.2.1     | 1      | LAN2                            |
| R2     | g0/1        | 2.1.1.1         | 2      | To R3                           |
| R2     | Loopback1   | 20.1.1.1        | 2      | Router ID                       |
| R3     | fg0/1       | 2.1.1.2         | 2      | To R2                           |
| R3     | Loopback1   | 30.1.1.1        | 2      | Router ID                       |
| R3     | Gi0/0       | 192.168.3.1     | 3      | LAN3                            |

---

## ⚙️ Key CLI Configuration

### 🔹 R1
```
router ospf 1
 network 192.168.1.0 0.0.0.255 area 0
 network 1.1.1.0 0.0.0.255 area 1
 area 1 virtual-link 20.1.1.1
```

### 🔹 R2
```
router ospf 1
 network 1.1.1.0 0.0.0.255 area 1
 network 20.1.1.1 0.0.0.0 area 2
 network 2.1.1.0 0.0.0.255 area 2
 network 192.168.2.0 0.0.0.255 area 1
 area 1 virtual-link 10.1.1.1
 area 2 virtual-link 30.1.1.1
```

### 🔹 R3
```
router ospf 1
 network 2.1.1.0 0.0.0.255 area 2
 network 30.1.1.1 0.0.0.0 area 2
 network 192.168.3.0 0.0.0.255 area 3
 area 2 virtual-link 20.1.1.1
```

> 🔑 **Syntax:** `area <TRANSIT-AREA> virtual-link <REMOTE ROUTER ID>`. The number after `area` is the **transit area** the link crosses, not the backbone. The address is the far router's **Router ID** (here, its loopback), **not** an interface IP.

---

## 🧭 Which Routers Can Form a Virtual Link?

A virtual link needs both endpoints to share a **common transit area**. Checking every pair in this topology:

| Pair    | Areas of each router                | Common area | Virtual link possible? | Used here |
|---------|----------------------------------------|---------------|----------------------------|-------------|
| R1 ↔ R2 | R1: {0, 1} · R2: {1, 2}                | **Area 1**     | ✅ Yes, transit = Area 1   | ✅ VL #1    |
| R2 ↔ R3 | R2: {1, 2} · R3: {2, 3}                | **Area 2**     | ✅ Yes, transit = Area 2   | ✅ VL #2    |
| R1 ↔ R3 | R1: {0, 1} · R3: {2, 3}                | **None**       | ❌ No direct virtual link  | Reached indirectly through R2 |

> R1 and R3 share no area, so they can't be linked directly. R3 reaches the backbone by **chaining** through R2: R3 → R2 (VL #2) → R1 (VL #1) → Area 0. And if all three routers had been placed in one single area, no virtual link would have been needed at all.

---

## 🔍 Verification & Evidence

### 1️⃣ Before the fix — OSPF flags the broken design
```
%OSPF-4-ERRRCV: Received invalid packet: mismatch area ID, from backbone area
must be virtual-link but not found
```
> This message is the tell-tale sign of a virtual link that is configured on **one end only**. The configured router starts sending backbone-area packets across the transit area, and the other router rejects them because it has no matching virtual link yet. It repeated on R2 and R3 until each of them configured its end.

### 2️⃣ R1 configures its end first — link is "up" but the adjacency isn't
```
R1(config-router)#area 1 virtual-link 20.1.1.1
R1#show ip ospf virtual-links
Virtual Link OSPF_VL0 to router 20.1.1.1 is up
  Run as demand circuit
  Transit area 1, via interface Serial0/0/0, Cost of using 64
  Transmit Delay is 1 sec, State POINT_TO_POINT,
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
  Adjacency State DOWN
```
> **"is up"** means OSPF found a working path to the remote router through the transit area (Area 1, over Serial0/0/0, cost 64). **"Adjacency State DOWN"** means the far end hasn't been configured yet, so no neighbor relationship exists.

### 3️⃣ Both ends configured — adjacency reaches FULL
```
R2(config-router)#area 1 virtual-link 10.1.1.1
%OSPF-5-ADJCHG: Process 1, Nbr 10.1.1.1 on OSPF_VL0 from LOADING to FULL, Loading Done

R3(config-router)#area 2 virtual-link 20.1.1.1
%OSPF-5-ADJCHG: Process 1, Nbr 20.1.1.1 on OSPF_VL0 from LOADING to FULL, Loading Done
```
> As soon as the second end of each virtual link was configured, the adjacency completed. The virtual link shows up as its own interface (`OSPF_VL0`, `OSPF_VL1`).

### 4️⃣ R2 — four adjacencies, two of them virtual
```
R2#show ip ospf neighbor
Neighbor ID   Pri  State     Dead Time  Address    Interface
30.1.1.1      1    FULL/DR   00:00:33   2.1.1.2    GigabitEthernet0/1
10.1.1.1      0    FULL/ -   00:00:31   1.1.1.1    Serial0/0/0
30.1.1.1      0    FULL/ -   00:00:38   2.1.1.2    OSPF_VL1
10.1.1.1      0    FULL/ -   00:00:39   1.1.1.1    OSPF_VL0
```
> R2 talks to each neighbor **twice**: once over the physical link and once over the virtual link. Virtual-link neighbors always show priority `0` and `FULL/ -`, since no DR/BDR election happens on a virtual link.

### 5️⃣ R2 — now an ABR in three areas
```
R2#show ip ospf
It is an area border router
Number of areas in this router is 3. 3 normal 0 stub 0 nssa
```
> R2 has physical interfaces in only two areas (1 and 2), yet it reports **three** areas. The third is the backbone, which R2 now belongs to through the virtual link.

### 6️⃣ Routing tables — proof of who is in which area

**R1:**
```
O IA  2.1.1.0/24      [110/65] via 1.1.1.2   (00:02:13)   ← Area 2
O IA  20.1.1.1/32     [110/65] via 1.1.1.2   (00:02:13)   ← Area 2
O IA  30.1.1.1/32     [110/66] via 1.1.1.2   (00:02:13)   ← Area 2
O     192.168.2.0/24  [110/65] via 1.1.1.2   (00:05:49)   ← Area 1 (R1's own area)
O IA  192.168.3.0/24  [110/66] via 1.1.1.2   (00:00:13)   ← Area 3
```

**R3:**
```
O IA  1.1.1.0/24      [110/65] via 2.1.1.1
O IA  10.1.1.1/32     [110/66] via 2.1.1.1
O     20.1.1.1/32     [110/2]  via 2.1.1.1
O     192.168.1.0/24  [110/66] via 2.1.1.1     ← Area 0's LAN, shown as plain "O"
O IA  192.168.2.0/24  [110/2]  via 2.1.1.1
```

> **Two important findings:**
>
> **1) R3 sees Area 0's LAN as plain `O` (intra-area), not `O IA`.** R3 has no interface in Area 0, yet it treats `192.168.1.0/24` as an intra-area route. That only happens because the virtual links made R3 a member of the backbone.
>
> **2) The route ages show the staged convergence.** On R1, the Area 2 routes are 2 minutes 13 seconds old, while `192.168.3.0/24` is only 13 seconds old. The first virtual link (R1–R2) delivered Area 2's routes, and Area 3's route arrived only after the second virtual link (R2–R3) came up.

---

## 📋 Rules & Caveats for Virtual Links

| Rule                                   | Detail                                                                    |
|------------------------------------------|------------------------------------------------------------------------------|
| Common transit area required             | Both endpoints must have an interface in the same non-backbone area           |
| Transit area can't be a stub area        | Virtual links can't cross stub areas (or NSSA)                                |
| Identified by Router ID                  | The `virtual-link` command uses the remote router's Router ID, not an IP      |
| Must be configured on both ends          | One-sided config produces the `ERRRCV … must be virtual-link` message         |
| Runs as a demand circuit                 | Hellos aren't refreshed periodically the way they are on a normal link        |
| Cost = path cost through the transit area | Seen as `Cost of using 64` (the Serial link across Area 1)                   |
| A workaround, not a design goal          | Chained virtual links are fragile; the proper fix is redesigning the areas so every one touches Area 0 |

---

## 🎯 Key Learnings

- OSPF requires every non-backbone area to connect to **Area 0**. When that isn't physically possible, a **virtual link** extends the backbone through a transit area.
- The `%OSPF-4-ERRRCV … backbone area must be virtual-link but not found` message means a virtual link is configured on only one end.
- A virtual link endpoint is specified by **Router ID**, and the area number in the command is the **transit area**, not Area 0.
- Both endpoints must share a common area: R1–R2 (Area 1) and R2–R3 (Area 2) qualify, while R1–R3 share none and can't link directly.
- Virtual links can be **chained**: R3 joined the backbone through R2, which itself joined through R1.
- `show ip ospf virtual-links` shows the transit area, the path cost, and the adjacency state, and "is up" with "Adjacency State DOWN" means the other end is still unconfigured.
- A router's real area membership shows in its routing table: R3 saw Area 0's LAN as `O`, not `O IA`, proving it had joined the backbone.
- Virtual links are a repair tool. In a real network, redesigning the areas is the cleaner long-term answer.

---

## ✅ Outcomes

- Built an OSPF design with two areas cut off from the backbone and captured the resulting errors
- Configured and verified two chained virtual links across two transit areas
- Confirmed adjacencies, virtual-link status, and area membership using neighbor tables and routing tables
- Determined valid virtual-link pairings from each router's area memberships
- Added a CCNP-level OSPF area-design lab to the portfolio, covering a core ENARSI Layer 3 topic

---

## 🛠️ Tools Used

- Cisco Packet Tracer
- Cisco IOS (Multi-Area OSPF, Virtual Links)
- `show ip ospf virtual-links`, `show ip ospf neighbor`, `show ip ospf`, `show ip route`

---

## 👤 Author

**Maaz Khan**
CCNA Certified | Network & NOC Engineer — pursuing CCNP Enterprise (ENCOR/ENARSI)
📍 Lower Dir, KPK, Pakistan
🔗 [LinkedIn](https://www.linkedin.com/in/maazkhanms) · [GitHub](https://github.com/maazkhanms)

---

⭐ If you found this lab useful, consider starring the repo — more OSPF, EIGRP, BGP, IPv6, and routing-fundamentals labs coming in this series!
