# Mastering Cisco BGP Weight

![Cisco BGP Weight: do you prefer this weight over value 0?](images/bgp-weight-cover.png)

> Study notes by [Nazirul Roslan](https://www.linkedin.com/in/nazirulroslan)

---

## 1. What is BGP Weight?

The BGP **Weight** is a **Cisco proprietary** value used to influence best path selection on a single router. It is the **first** attribute checked in Cisco's BGP decision process.

- **Locally significant:** Weight only affects the router it is configured on. It is **NOT advertised** in BGP updates to any peer.
- **Similar to administrative distance:** think of it as a local preference knob. If Router A sets a weight, Router B never learns about it.
- **Purpose:** a simple, router-local way to prefer one path over another when several paths to the same destination exist.

---

## 2. Key Characteristics

| Property | Value |
|---|---|
| Preference | **Higher is better** |
| Default (learned from any BGP peer, eBGP or iBGP) | `0` |
| Default (locally originated via `network` or `redistribute`) | `32768` |
| Range | `0` – `65535` |
| Vendor | Cisco specific |
| Carried in updates? | **No.** Not a true path attribute, just a local parameter |

Because locally originated routes get `32768`, they are strongly preferred by default.

---

## 3. Configuration Methods

### Method 1: Per neighbor

Applies a weight to **all routes** learned from one neighbor.

```
router bgp <AS_NUMBER>
 neighbor <NEIGHBOR_IP> weight <WEIGHT_VALUE>
```

Example: `neighbor 10.1.1.2 weight 100` makes routes from 10.1.1.2 preferred over routes from another neighbor that still has the default weight of 0.

### Method 2: Route map (granular)

Sets weight only for routes that match specific criteria (prefixes, AS paths, and so on).

```
! Define what to match
ip prefix-list MY_PREFIXES permit 172.16.0.0/16
!
route-map SET_MY_WEIGHT permit 10
 match ip address prefix-list MY_PREFIXES
 set weight <WEIGHT_VALUE>
!
router bgp <AS_NUMBER>
 neighbor <NEIGHBOR_IP> route-map SET_MY_WEIGHT in
```

Routes matching `MY_PREFIXES` from that neighbor get the weight applied.

> ⚠️ This route map has no catch-all sequence, so every route that does **not** match `MY_PREFIXES` is dropped by the implicit deny. Add an empty `route-map SET_MY_WEIGHT permit 20` if you only want to set weight, not filter.

---

## 4. Common Use Cases

### Use case 1: Prefer one eBGP neighbor for all routes

R1 has two eBGP neighbors, R2 (`10.0.0.2`) and R3 (`10.0.0.3`), and should prefer R2 for everything.

```
! On Router R1 (AS 65001)
router bgp 65001
 neighbor 10.0.0.2 remote-as 65002
 neighbor 10.0.0.2 weight 200   ! Higher weight for R2
 neighbor 10.0.0.3 remote-as 65003
 ! R3 routes keep the default weight 0
```

**Effect:** R1 prefers paths from R2 (weight 200) over paths from R3 (weight 0) for the same destinations.

### Use case 2: Prefer specific prefixes from a neighbor

R1 should prefer R2 (`10.0.0.2`) only for prefixes inside `172.16.0.0/12`.

```
! On Router R1 (AS 65001)
ip prefix-list PREFER_THESE permit 172.16.0.0/12 le 32
!
route-map PREFER_R2_FOR_SPECIFIC permit 10
 match ip address prefix-list PREFER_THESE
 set weight 150
route-map PREFER_R2_FOR_SPECIFIC permit 20
 ! Other routes from R2 pass with default weight 0
!
router bgp 65001
 neighbor 10.0.0.2 remote-as 65002
 neighbor 10.0.0.2 route-map PREFER_R2_FOR_SPECIFIC in
```

**Effect:** only `172.16.0.0/12` and its more-specifics from R2 get weight 150. Everything else from R2 stays at 0.

---

## 5. Limitations & Best Practices

### Limitations

- **Local significance only:** the biggest limitation. To influence other routers in your AS or in other ASes, weight is the wrong tool.
- **Scalability:** configuring weight on many routers for a consistent policy is cumbersome and error-prone.

### Best practices

- **Primary use:** influencing one router, for example a multi-homed edge router that should prefer one ISP link.
- **Route maps for granularity:** use `set weight` when the weight should depend on prefix, AS path, or other attributes.
- **Use the right tool for wider influence:**

| Goal | Tool |
|---|---|
| One router's outbound choice | **Weight** |
| Whole AS outbound choice | **Local Preference** (higher is better, shared over iBGP) |
| Inbound traffic from other ASes | **AS-Path prepending** or **MED** |

- **Document it:** because weight is invisible to other routers, record where and why you use it.

---

## 6. Troubleshooting Toolkit

```
show ip bgp
```
Shows the BGP table with a **Weight** column. `>` marks the best path.

```
show ip bgp <PREFIX>
```
All paths to one prefix with their attributes (weight, local preference, AS path, and so on). Confirms whether weight is applied and deciding the best path.

```
show route-map [MAP_NAME]
```
Shows the route-map configuration and, depending on the IOS version, match counters.

```
debug ip bgp updates [neighbor-ip] [in|out]
```
Shows updates being processed. Weight isn't inside the update, but you can see which routes hit your policy. Use with caution in production.

```
clear ip bgp * soft [in|out]
```
Re-applies policy changes (such as a new weight route map) without resetting sessions. Use `in` for inbound policy.
