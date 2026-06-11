# Static Routing with Backup Links — GNS3 Lab

A hands-on walkthrough of building a resilient multi-router network in GNS3 using static routing and administrative distance-based failover. Three routers, four LANs, eight PCs — and deliberate link failures to prove the backup paths actually work.

---

## What This Covers

- Assigning IP addresses across a 3-router topology in GNS3
- Writing static routes with primary and backup paths using administrative distance
- Shutting down links and verifying traffic automatically reroutes
- Understanding why both ends of a link must go down for clean failover

---

## Network Topology

![Network Topology](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/412534iuh5lhojib232c.png)

| Device | Interface | IP Address |
|---|---|---|
| RouterA | f0/0 | 10.10.20.250/24 |
| RouterA | f2/0 | 10.10.100.1/24 |
| RouterA | f3/1 | 10.10.200.1/24 |
| RouterB | f2/0 | 10.10.200.2/24 |
| RouterB | f0/0 | 10.10.150.1/24 |
| RouterB | f3/1 | 10.10.1.250/24 |
| RouterB | f3/0 | 10.10.10.250/24 |
| RouterC | f0/0 | 10.10.100.2/24 |
| RouterC | f2/0 | 10.10.150.2/24 |
| RouterC | f3/0 | 10.10.5.250/24 |
| PC1 | e0 | 10.10.20.1/24 |
| PC2 | e0 | 10.10.20.2/24 |
| PC3 | e0 | 10.10.10.1/24 |
| PC4 | e0 | 10.10.10.2/24 |
| PC5 | e0 | 10.10.1.1/24 |
| PC6 | e0 | 10.10.1.2/24 |
| PC7 | e0 | 10.10.5.1/24 |
| PC8 | e0 | 10.10.5.2/24 |

**Link 1** = RouterA f2/0 ↔ RouterC f0/0 (10.10.100.0/24)  
**Link 2** = RouterB f0/0 ↔ RouterC f2/0 (10.10.150.0/24)

---

## Router IP Configuration

### RouterA

```
RouterA# enable
RouterA# configure terminal
RouterA(config)# interface f0/0
RouterA(config-if)# ip address 10.10.20.250 255.255.255.0
RouterA(config-if)# no shutdown
RouterA(config-if)# exit
RouterA(config)# interface f2/0
RouterA(config-if)# ip address 10.10.100.1 255.255.255.0
RouterA(config-if)# no shutdown
RouterA(config-if)# exit
RouterA(config)# interface f3/1
RouterA(config-if)# ip address 10.10.200.1 255.255.255.0
RouterA(config-if)# no shutdown
RouterA(config-if)# end
RouterA# write memory
```

![RouterA IP Configuration](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/y05u7njov89hvyegs7j9.png)

### RouterB

```
RouterB# enable
RouterB# configure terminal
RouterB(config)# interface f2/0
RouterB(config-if)# ip address 10.10.200.2 255.255.255.0
RouterB(config-if)# no shutdown
RouterB(config-if)# exit
RouterB(config)# interface f0/0
RouterB(config-if)# ip address 10.10.150.1 255.255.255.0
RouterB(config-if)# no shutdown
RouterB(config-if)# exit
RouterB(config)# interface f3/1
RouterB(config-if)# ip address 10.10.1.250 255.255.255.0
RouterB(config-if)# no shutdown
RouterB(config-if)# exit
RouterB(config)# interface f3/0
RouterB(config-if)# ip address 10.10.10.250 255.255.255.0
RouterB(config-if)# no shutdown
RouterB(config-if)# end
RouterB# write memory
```

![RouterB IP Configuration](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/govz7nnca7rh9fd31mbz.png)

### RouterC

```
RouterC# enable
RouterC# configure terminal
RouterC(config)# interface f0/0
RouterC(config-if)# ip address 10.10.100.2 255.255.255.0
RouterC(config-if)# no shutdown
RouterC(config-if)# exit
RouterC(config)# interface f2/0
RouterC(config-if)# ip address 10.10.150.2 255.255.255.0
RouterC(config-if)# no shutdown
RouterC(config-if)# exit
RouterC(config)# interface f3/0
RouterC(config-if)# ip address 10.10.5.250 255.255.255.0
RouterC(config-if)# no shutdown
RouterC(config-if)# end
RouterC# write memory
```

![RouterC IP Configuration](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xkxkfmb24oke2fyzj9kv.png)

---

## PC IP Configuration

Each VPCS device gets its IP, subnet mask, and default gateway pointing to the connected router's LAN interface.

```
# PC1 example
PC1> ip 10.10.20.1/24 10.10.20.250
PC1> save
```

| PC1 | PC2 |
|---|---|
| ![PC1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5eglaiomil0tiaae3dww.png) | ![PC2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mcsd27tqtvokf33lj5ak.png) |

| PC3 | PC4 |
|---|---|
| ![PC3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/la4truegzjdggnwp0h07.png) | ![PC4](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3lvr48ghni89i0otcwr3.png) |

| PC5 | PC6 |
|---|---|
| ![PC5](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/63a34pxyc5rjy6i6tnxi.png) | ![PC6](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/obtbkolwxbv32drlew6q.png) |

| PC7 | PC8 |
|---|---|
| ![PC7](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xbgo3sh21ba16hozgsmh.png) | ![PC8](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/my8oopbvhwh22zuxrtoq.png) |

---

## Static Route Configuration (Primary + Backup)

The key concept: primary routes use the default AD of **1**, backup routes are given AD of **10**. The router installs only the primary. When that link goes down, the backup activates automatically.

```
# Syntax
ip route [network] [mask] [next-hop]        ← primary (AD = 1)
ip route [network] [mask] [next-hop] 10     ← backup  (AD = 10)
```

### RouterA

```
RouterA# configure terminal
RouterA(config)# ip route 10.10.10.0 255.255.255.0 10.10.200.2
RouterA(config)# ip route 10.10.10.0 255.255.255.0 10.10.100.2 10
RouterA(config)# ip route 10.10.1.0 255.255.255.0 10.10.200.2
RouterA(config)# ip route 10.10.1.0 255.255.255.0 10.10.100.2 10
RouterA(config)# ip route 10.10.5.0 255.255.255.0 10.10.200.2
RouterA(config)# ip route 10.10.5.0 255.255.255.0 10.10.100.2 10
RouterA(config)# end
RouterA# write memory
```

![RouterA Static Routes](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/llf3whwzpf5d5r7er7wb.png)

### RouterB

```
RouterB# configure terminal
RouterB(config)# ip route 10.10.20.0 255.255.255.0 10.10.200.1
RouterB(config)# ip route 10.10.20.0 255.255.255.0 10.10.150.2 10
RouterB(config)# ip route 10.10.5.0 255.255.255.0 10.10.150.2
RouterB(config)# ip route 10.10.5.0 255.255.255.0 10.10.200.1 10
RouterB(config)# ip route 10.10.100.0 255.255.255.0 10.10.150.2
RouterB(config)# ip route 10.10.100.0 255.255.255.0 10.10.200.1 10
RouterB(config)# end
RouterB# write memory
```

![RouterB Static Routes](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/raskmwgc3tvwu5kp2cu0.png)

### RouterC

```
RouterC# configure terminal
RouterC(config)# ip route 10.10.20.0 255.255.255.0 10.10.100.1
RouterC(config)# ip route 10.10.20.0 255.255.255.0 10.10.150.1 10
RouterC(config)# ip route 10.10.10.0 255.255.255.0 10.10.150.1
RouterC(config)# ip route 10.10.10.0 255.255.255.0 10.10.100.1 10
RouterC(config)# ip route 10.10.1.0 255.255.255.0 10.10.150.1
RouterC(config)# ip route 10.10.1.0 255.255.255.0 10.10.100.1 10
RouterC(config)# end
RouterC# write memory
```

![RouterC Static Routes](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/amwfv7sio9ddsiyr6c4r.png)

---

## Baseline Connectivity

Before testing failover, verify full end-to-end reachability with all links up.

![PC1 to PC8 Baseline](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/es6m41wt0jsdoy3yzo0p.png)

![PC2 to PC5 Baseline](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p0fb2jkd4dhhsaj6cwpr.png)

---

## Scenario 1 — Ping PC8 from PC1 with Link 1 Down

Shut down **both ends** of Link 1 (RouterA f2/0 and RouterC f0/0). Traffic reroutes through RouterB.

```
RouterA(config)# interface f2/0
RouterA(config-if)# shutdown

RouterC(config)# interface f0/0
RouterC(config-if)# shutdown
```

![RouterA Link 1 Shutdown](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5goi9hf3l52khw297t84.png)

![RouterC Link 1 Shutdown](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7qpltt4kylj8hw8p6eqk.png)

**Result — PC1 still reaches PC8 via backup path:**

![PC1 to PC8 with Link 1 Down](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4paukm2j9afpihwifxj3.png)

---

## Scenario 2 — Ping PC7 from PC3 with Link 2 Down (Link 1 Active)

Restore Link 1, then shut down **both ends** of Link 2 (RouterC f2/0 and RouterB f0/0). Traffic flows through Link 1.

```
RouterC(config)# interface f2/0
RouterC(config-if)# shutdown

RouterB(config)# interface f0/0
RouterB(config-if)# shutdown
```

![RouterC Link 2 Shutdown](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jsos9xrodrwjyz0ajcuq.png)

![RouterB Link 2 Shutdown](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1io4k7b7gkm51m625z43.png)

**Result — PC3 still reaches PC7 via Link 1:**

![PC3 to PC7 with Link 2 Down](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0ymca4t509jy1vspmdbp.png)

---

## Verification Commands

```
# View routing table — backup routes show as [10/0]
RouterA# show ip route

# Check interface status
RouterA# show ip interface brief

# Detailed interface info
RouterA# show interfaces f2/0
```

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Only shutting down one side of a link | Intermittent packet loss, no clean failover | Shut down both interfaces |
| No AD on backup route | Router load-balances instead of using standby | Add `10` at end of backup route command |
| Same AD on both routes | Unpredictable routing behavior | Primary = 1 (default), backup = 10 |
| Missing return routes | One-way traffic, pings fail | Configure routes in both directions on all routers |
| Skipping `write memory` | Config lost on reload | Always save after configuring |

---

## Read the Full Article

For the complete walkthrough with explanation of every concept:  
📖 [Static Routing with Backup Links — Dev.to](https://dev.to/almahmudkhalif/)

---

## 🌐 Connect With Me

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?logo=linkedin)](https://www.linkedin.com/in/almahmudkhalif/)
[![Dev.to](https://img.shields.io/badge/Dev.to-Articles-black?logo=devdotto)](https://dev.to/almahmudkhalif/)
