# NetPractice

> *This project has been created as part of the 42 curriculum by jboustan*

A hands-on introduction to TCP/IP networking. Ten broken network topologies must be repaired by configuring IP addresses, subnet masks, default gateways and routing tables until every communication goal passes.

This repository contains the exported configuration for each level, plus a **full written explanation of every level** — not just the answers, but the reasoning that produces them.

---

## Table of Contents

- [Description](#description)
- [Repository Structure](#repository-structure)
- [Instructions](#instructions)
- [Core Concepts](#core-concepts)
  - [1. Anatomy of an IPv4 address](#1-anatomy-of-an-ipv4-address)
  - [2. The subnet mask](#2-the-subnet-mask)
  - [3. CIDR notation](#3-cidr-notation)
  - [4. Finding the network address](#4-finding-the-network-address)
  - [5. The block size shortcut](#5-the-block-size-shortcut)
  - [6. Mask reference table](#6-mask-reference-table)
  - [7. Reserved and special addresses](#7-reserved-and-special-addresses)
  - [8. Switches vs routers](#8-switches-vs-routers)
  - [9. Default gateway](#9-default-gateway)
  - [10. Routing tables](#10-routing-tables)
  - [11. Bidirectional routing](#11-bidirectional-routing)
- [The Solving Method](#the-solving-method)
- [Level Index](#level-index)
- [Level Walkthroughs](#level-walkthroughs)
- [Error Messages and What They Mean](#error-messages-and-what-they-mean)
- [Common Pitfalls Checklist](#common-pitfalls-checklist)
- [Resources](#resources)
- [Use of AI](#use-of-ai)
- [Author](#author)
- [License](#license)

---

## Description

**NetPractice** is a networking exercise from the 42 school curriculum. It consists of 10 progressively harder levels, each presenting a broken network topology displayed in an interactive browser-based simulator. The goal is to configure IP addresses, subnet masks, and routing table entries so that all communication goals shown for that level are satisfied.

The exercise covers the practical fundamentals of TCP/IP networking: how addresses are structured, how subnet masks determine network boundaries, how routers forward packets between networks, and how routing tables are read and written. Each level introduces new constraints — additional routers, ISP-assigned address ranges, public vs. private IP restrictions — that require applying the concepts learned in previous levels.

The difficulty curve is deliberate:

| Levels | Focus |
|---|---|
| 1–3 | Addressing and masks on a single switched network |
| 4 | Routers, and fitting a network into a gap between fixed networks |
| 5–6 | Gateways, routing tables, and the Internet as a node |
| 7 | Two chained routers, three subnets, IP conflicts |
| 8 | ISP-assigned blocks, nested subnetting (/26 → /28) |
| 9–10 | Public vs. private addressing, full address-space planning |

---

## Repository Structure

```
.
├── README.md
├── levels/                  # Written explanation of each level
│   ├── level1.md
│   ├── level2.md
│   ├── ...
│   └── level10.md
├── net_practice.1.7/        # The training interface
└── *.config                 # The 10 exported level configurations (evaluated)
```

---

## Instructions

### Running the training interface

1. Clone or download the repository.
2. Navigate to the `net_practice.1.7/` folder.
3. Open `index.html` in any modern web browser (no server required).
4. Enter your login when prompted to load the level interface.
5. Select a level from the dropdown and configure the editable fields until all goals show **OK**.

### Reading the interface

- **Pink / shaded fields** are fixed. They cannot be changed, and they are your constraints.
- **White fields** are editable. These are what you fix.
- The **log panel** reports exactly where and why a packet failed. Read it before guessing.

### Exporting a configuration

Once all goals for a level pass, click **"Get my config"**. This downloads a configuration file for that level. Repeat for all 10 levels.

### Submission requirements

- Complete all 10 levels in the training interface.
- Export one configuration file per level using **"Get my config"**.
- Place all 10 exported configuration files at the **root of the repository** (alongside this README).
- Push the repository. The 10 config files at the root are what is evaluated.

---

## Core Concepts

Everything in NetPractice reduces to a small number of rules. This section is the complete theory needed to solve all ten levels.

### 1. Anatomy of an IPv4 address

An IPv4 address is 32 bits, written as four decimal **octets** separated by dots:

```
104   .   97   .   23   .   12
 ↑         ↑        ↑        ↑
octet 1  octet 2  octet 3  octet 4
```

Each octet must be between **0 and 255**. An address such as `104.93.23.378` is invalid on its face, because `378` exceeds 255. Levels 1 and 3 both open with this trap.

### 2. The subnet mask

The mask splits an address into a **network part** and a **host part**:

- Network part → identifies which network the device belongs to
- Host part → identifies the specific device inside that network

The postal analogy: the network is the street, the host is the house number. Two devices can talk **directly** only if they are on the same street.

In the mask, bits set to `1` mark the network part and bits set to `0` mark the host part. Because the 1s are always contiguous and come first, only nine octet values are ever valid in a mask:

```
0, 128, 192, 224, 240, 248, 252, 254, 255
```

Anything else — `255.255.255.32`, seen in Level 2 — is not a legal mask.

### 3. CIDR notation

`/24` is shorthand for a mask, and the number is simply **how many 1 bits the mask contains**:

```
255  =  11111111   →  8 ones
255  =  11111111   →  8 ones
255  =  11111111   →  8 ones
0    =  00000000   →  0 ones
                      ──────
                      24  →  /24
```

So `104.97.23.0/24` and `104.97.23.0 / 255.255.255.0` mean exactly the same thing. Both forms appear throughout NetPractice, sometimes on the same screen.

### 4. Finding the network address

Apply the mask to the IP with a **bitwise AND**: output 1 only where both bits are 1.

```
IP last octet:    125  =  0 1 1 1 1 1 0 1
Mask last octet:  128  =  1 0 0 0 0 0 0 0
                  AND  =  0 0 0 0 0 0 0 0  =  0
```

So `104.198.249.125` with `/25` is on network `104.198.249.0`. Only the octets where the mask is neither 0 nor 255 need this work — octets where the mask is 255 are copied as-is, and octets where the mask is 0 become 0.

This single operation answers the question that decides almost every level: **are these two devices on the same network?** Compute AND(IP, mask) for both. Same result → same network.

### 5. The block size shortcut

Binary AND is exact but slow. For any mask, this is faster:

```
block size  = 256 − (last non-255 octet of the mask)
block index = floor(that octet of the IP ÷ block size)
block start = block index × block size
block end   = block start + block size − 1
```

Worked example, `/27` (mask `255.255.255.224`) with an IP ending in `.222`:

```
block size  = 256 − 224 = 32
block index = 222 ÷ 32 = 6.9  →  6
block start = 6 × 32 = 192
block end   = 192 + 31 = 223

→ Network .192 | Usable .193–.222 | Broadcast .223
```

The **first** address in every block is the network address and the **last** is the broadcast address. Neither can be assigned to a device, which is why a block of size *n* gives *n − 2* usable hosts.

### 6. Mask reference table

| CIDR | Mask | Block size | Usable hosts | Boundaries in the affected octet |
|---|---|---|---|---|
| /8 | 255.0.0.0 | — | 16,777,214 | first octet only |
| /16 | 255.255.0.0 | — | 65,534 | first two octets |
| /18 | 255.255.192.0 | 64 (3rd octet) | 16,382 | 0, 64, 128, 192 |
| /24 | 255.255.255.0 | 256 | 254 | 0 |
| /25 | 255.255.255.128 | 128 | 126 | 0, 128 |
| /26 | 255.255.255.192 | 64 | 62 | 0, 64, 128, 192 |
| /27 | 255.255.255.224 | 32 | 30 | 0, 32, 64, 96, 128, 160, 192, 224 |
| /28 | 255.255.255.240 | 16 | 14 | 0, 16, 32, 48, … |
| /29 | 255.255.255.248 | 8 | 6 | 0, 8, 16, 24, … |
| /30 | 255.255.255.252 | 4 | **2** | 0, 4, 8, 12, … |
| /31 | 255.255.255.254 | 2 | **0** | every even number |

Two entries deserve attention:

- **/30 is the smallest usable subnet.** Four addresses: one network, two hosts, one broadcast. This is exactly right for a router-to-router link, which needs one address per side — and that is how Levels 9 and 10 use it.
- **/31 has no usable hosts at all.** It appears in Levels 6 and 10 as a deliberately wrong value, a route destination that looks plausible but covers only two addresses.

### 7. Reserved and special addresses

| Range | Name | Why it matters |
|---|---|---|
| `127.0.0.0/8` | Loopback | A device talking to itself. Never valid for host-to-host communication. Traps in Levels 2 and 3. |
| `10.0.0.0/8` | Private (RFC 1918) | Not routed over the Internet. |
| `172.16.0.0/12` | Private (RFC 1918) | Not routed over the Internet. |
| `192.168.0.0/16` | Private (RFC 1918) | Not routed over the Internet. |

**Why private ranges exist:** there are not enough IPv4 addresses for every device on Earth, so internal networks reuse the same private blocks. Your home router may use `192.168.1.1`, and so may millions of others. Because the same private address exists in thousands of networks simultaneously, the Internet cannot know which one a packet is meant for — so it refuses to route them at all.

**The practical consequence in NetPractice:** a host with a private IP may successfully send a packet *out* to the Internet, but the reply can never come back. The log reports `private subnets not routed over Internet`. Any host with an Internet goal needs a **public** address. Levels 9 and 10 are built entirely around this rule.

### 8. Switches vs routers

A **switch** connects devices within one network. It does not join networks. Every device on a switch must therefore share the same network address *and* the same mask.

A **router** connects different networks. Each of its interfaces sits on a **different, non-overlapping network** — this is mandatory. If two interfaces of one router are on the same subnet, the router has no basis for deciding which way to forward a packet, and routing breaks entirely (the root cause of the failure in Level 7).

A router also **automatically knows** how to reach any network it is directly connected to. No route entry is needed for those. Route entries are only required for networks that sit *behind* another router.

### 9. Default gateway

When a destination lies outside a host's own network, the host cannot deliver the packet itself. It hands it to a router — the **default gateway**.

The gateway is the router's IP **on the host's own network**. This produces the single most-used verification in the whole project:

```
AND(gateway, mask)  must equal  AND(interface IP, mask)
```

If the gateway is not on your subnet, you cannot reach it, and nothing else about the configuration matters. You cannot hand your letter to a post office on a different street.

### 10. Routing tables

A route is a rule with two halves:

```
[destination]  =>  [gateway]
      ↑                ↑
 what traffic      where to send it
 this rule covers
```

- **`default`** and **`0.0.0.0/0`** are the same thing: match any destination. `/0` means zero network bits, so nothing is constrained.
- A **specific route** such as `10.0.0.0/8` matches only `10.x.x.x`. If the actual destination is `160.194.242.253`, the rule never fires and the packet is dropped. This is the recurring bug in Levels 5, 6, 9 and 10.
- **Longest prefix match:** when several routes match, the most specific (longest prefix) wins. A `/28` route beats a `/26` route for addresses inside the `/28`.

A router checks its **directly connected networks first**. Only when no interface matches does it fall back to its routing table. This is why two routers can both hold a default route pointing at each other without creating a loop (explained in Level 7).

**When *not* to use `default`:** the Internet node must never have a default route pointing back into a private network. Two devices with default routes aimed at each other bounce unknown packets back and forth until they are discarded. The Internet needs an explicit route naming the specific subnet it should reach back into — the reasoning is worked through in Level 6.

### 11. Bidirectional routing

Communication requires a working path in **both directions**. A packet that reaches its destination but whose reply cannot return is still a failed goal.

This is why the Internet node has its own routing table in Levels 6, 8, 9 and 10. It must be told which subnet to send replies to. A missing or too-narrow destination there produces a "no reverse way" failure even though the outbound path is perfect.

---

## The Solving Method

The same procedure solves every level, and it becomes essential from Level 8 onward.

**1. Read the goals.** They define what must work. Nothing else matters.

**2. Inventory fixed vs editable fields.** Write them down. The fixed values are not obstacles — they are the clues that determine everything else.

**3. Extract every constraint the fixed values impose, in this order:**

| Fixed thing | What it tells you |
|---|---|
| A fixed IP + mask pair | The exact network that link must use |
| A fixed mask alone | The block size, so the boundaries of whatever network you choose |
| A fixed **gateway** | The IP that a neighbouring interface **must** be assigned |
| A fixed **route destination** | The address range all relevant host IPs **must** fall inside |

The third row is easy to miss and decisive. In Level 8, `Host D`'s fixed gateway `100.11.25.171` is the only thing that tells you what R23's IP must be. In Level 10, `H4`'s fixed gateway `.129` fixes R23 the same way.

**4. Map the address space.** For levels sharing one `/24` between several subnets, draw the last octet as a line and mark what is taken. The gaps are where new subnets go.

```
.0──────────.127   .128─────.191   .192─────.223   .224──.251   .252──.255
   Network 3         Net 5           Net 4          unused        Net 2
```

**5. Assign the editable fields**, working outward from the constraints.

**6. Verify every pair.** For each link, AND(IP, mask) must match on both sides. For each router, all interfaces must land on different networks.

**7. Verify routing in both directions** for every goal, including the reply path.

---

## Level Index

| Level | Core challenge | New concepts | Walkthrough |
|---|---|---|---|
| 1 | Two switched networks, invalid octets | IP structure, masks, switches, network/broadcast | [level1.md](levels/level1.md) |
| 2 | Mixed editable IP/mask fields | /27, /30, invalid mask values, loopback, block size method | [level2.md](levels/level2.md) |
| 3 | Three hosts on one switch | /25, bitwise AND, two fixed anchors defining one network | [level3.md](levels/level3.md) |
| 4 | First router; fitting into a gap | Router interface isolation, /26, working around fixed networks | [level4.md](levels/level4.md) |
| 5 | Two networks, one router | Default gateway, routing tables, default route, /18 | [level5.md](levels/level5.md) |
| 6 | The Internet as a node | `0.0.0.0/0`, /28, /31, bidirectional routing, why the Internet has no default | [level6.md](levels/level6.md) |
| 7 | Two chained routers | Three subnets, IP conflicts, why routers don't loop | [level7.md](levels/level7.md) |
| 8 | ISP-assigned block | Nested subnetting /26→/28, fixed gateway as a clue, longest prefix match | [level8.md](levels/level8.md) |
| 9 | Six goals, two routers, Internet | Public vs private IPs, orphaned gateways, /30 router links | [level9.md](levels/level9.md) |
| 10 | Seven goals, full address planning | Address space mapping, one supernet route, /27, /31 trap | [level10.md](levels/level10.md) |

---

## Level Walkthroughs

Each level below is summarised in a few lines. The linked file contains the full derivation: binary breakdowns, every calculation, why each wrong value was wrong, verification tables and a traced data path for every goal.

### Level 1 — Addressing basics

Two independent switched networks. Two hosts have invalid octets (`378`, `341`) and sit on the wrong networks. The fixed hosts define the networks; the editable IPs must join them.

**Lesson:** octets are 0–255, devices on a switch share one network, and the first and last addresses of a block are reserved.

### Level 2 — Masks, blocks and loopback

Introduces separately editable IP and mask fields, an illegal mask (`255.255.255.32`), and two hosts using loopback addresses. Also introduces the block size method that makes every later level tractable.

**Lesson:** mask octets have only nine legal values, `/30` fits exactly two hosts, and `127.x.x.x` can never be used between devices.

### Level 3 — Two anchors, one network

Three hosts share a switch. A fixed IP on one interface and a fixed mask on another **together** define the network everyone must join.

**Lesson:** when the constraints are spread across different interfaces, combine them first, then make everything else conform.

### Level 4 — The first router

A router's other two interfaces are fixed on `/25` and `/26` networks, leaving a 64-address gap between `.128` and `.191`. The fixed host IP `.132` sits inside that gap, so the switch network must be exactly `91.51.110.128/26`.

**Lesson:** router interfaces must never overlap. When neighbouring networks are fixed, find the gap and size your mask to fit it.

### Level 5 — Gateways and routes

Two networks either side of one router. Beyond fixing addresses, the host's default gateway and its route entry must both be corrected — the original route pointed at `10.0.0.0/8` via a gateway that wasn't even reachable.

**Lesson:** a gateway must live on the sender's own subnet, and a route destination must actually contain the target address. Use `default` to cover everything.

### Level 6 — The Internet appears

The Internet becomes a node with its own routing table. The webserver's outbound path works, but the Internet's route destination `60.160.208.0/31` covers only two addresses and excludes the server at `.227`, so replies are dropped.

**Lesson:** routing is bidirectional. Also, the Internet must never hold a default route back into a private network — two defaults pointing at each other create a loop.

### Level 7 — Two routers, three subnets

All six interfaces sit on one `/24`, and two devices share the IP `.1` — producing the distinctive "correct IP reached but wrong host" error. Splitting the space into `/26` blocks creates the three networks the topology requires.

**Lesson:** chained routers always need three networks — near side, router-to-router link, far side. Duplicate IPs cause wrong-host errors, not silent failures.

### Level 8 — Working inside an ISP block

The Internet's fixed route destination `146.244.132.0/26` dictates that **every** host address must fall inside that 64-address block, which is then subdivided into four `/28` networks. R2's fixed gateway `.62` reveals what R13's IP must be.

**Lesson:** work strictly from fixed values inward. A fixed gateway is an assignment instruction in disguise.

### Level 9 — Public addressing

Six goals across two routers and the Internet. The original configuration puts several hosts on `192.168.x.x` and `10.x.x.x`; the outbound path works but the Internet cannot reply, logging `private subnets not routed over Internet`.

**Lesson:** any host with an Internet goal needs a public address. And after changing a router interface IP, every gateway pointing at the old value must be updated or it silently orphans.

### Level 10 — Full address-space planning

Seven goals and mostly fixed fields. All four hosts must fit inside one supernet because the Internet has a single editable route destination — currently `/31`, covering two addresses and none of the hosts. Mapping the used ranges of `169.94.160.x` reveals the free gap `.192–.251` for the remaining subnet.

**Lesson:** map the whole address space before assigning anything, and design host addresses so a single route can cover them all.

---

## Error Messages and What They Mean

| Message | Meaning | Where to look |
|---|---|---|
| `no forward way` | No route matches the destination, or the gateway is unreachable | Route destination too narrow; gateway not on sender's subnet |
| `no reverse way` | The outbound path works, the reply path does not | The far side (often the Internet) lacks a route back to this subnet |
| `no interface for gateway X` | Nothing in the topology has IP `X` | A router interface was changed without updating the gateways pointing at it |
| `private subnets not routed over Internet` | A host with an Internet goal has an RFC 1918 address | Change that host — and its whole subnet — to public addressing |
| `correct IP reached but wrong host` | Two devices share the same IP | Duplicate address on the same network |

The log names the device where the packet died. Start reading there and work backwards.

---

## Common Pitfalls Checklist

Before assuming a level is broken, check every line:

- [ ] Every octet is 0–255
- [ ] Every mask octet is one of `0, 128, 192, 224, 240, 248, 252, 254, 255`
- [ ] Both ends of every link use the **same mask** — disagreeing about the boundary breaks the link even when the addresses look compatible
- [ ] No device is assigned a network address or a broadcast address
- [ ] No two devices on the same network share an IP
- [ ] Every router's interfaces are on different, non-overlapping subnets
- [ ] Every gateway satisfies `AND(gateway, mask) == AND(interface IP, mask)`
- [ ] Every route destination actually contains the address it needs to match
- [ ] No loopback (`127.x.x.x`) address is used between devices
- [ ] Every Internet-facing host has a public address
- [ ] After changing any interface IP, every gateway that referenced the old value was updated
- [ ] Both directions of every goal have been traced, not just the outbound one

---

## Resources

### Reference links

- [Microsoft Learn — TCP/IP addressing and subnetting](https://learn.microsoft.com/en-us/troubleshoot/windows-client/networking/tcpip-addressing-and-subnetting) — IP structure, subnet masks, and how a host decides whether a destination is local or remote
- [Computer Networking Notes — Subnetting Tutorial with Examples](https://www.computernetworkingnotes.com/ccna-study-guide/subnetting-tutorial-subnetting-explained-with-examples.html) — step-by-step subnetting with block calculations and usable host ranges
- [NetworkAcademy.io — The Subnet Mask](https://www.networkacademy.io/ccna/ip-subnetting/the-subnet-mask) — how masks work in binary and how CIDR prefixes map to dotted-decimal
- [Cloudflare — What is the OSI Model?](https://www.cloudflare.com/learning/ddos/glossary/open-systems-interconnection-model-osi/) — the seven layers and where IP sits
- [GeeksforGeeks — Default Gateway in Networking](https://www.geeksforgeeks.org/computer-networks/default-gateway-in-networking/) — what a gateway is and how routing decisions are made
- [Cisco — Switch vs Router](https://www.cisco.com/site/us/en/learn/topics/small-business/network-switch-vs-router.html) — Layer 2 switches versus Layer 3 routers
- [IETF — RFC 1918: Address Allocation for Private Internets](https://datatracker.ietf.org/doc/html/rfc1918) — the specification defining the three private ranges

### Concepts covered across the ten levels

IP address structure · subnet masks · bitwise AND · CIDR notation · network and broadcast addresses · subnetting and address space planning · block size calculation · default gateways · routers and interface isolation · switches · routing tables · default routes · longest prefix match · bidirectional routing · private vs public addressing (RFC 1918) · loopback addresses · the OSI model · the TCP/IP model

---

## Use of AI

Claude (Anthropic) was used as a learning companion throughout this project. All network configurations, calculations, and solutions were worked out independently. Claude served as a guide to clarify concepts, confirm understanding, and explain the reasoning behind networking rules — similar to asking a tutor to check your work or break down a topic when something was unclear.

---

## Author

**jboustan** — 42 School Student

---

## License

This project is part of the 42 curriculum.
