# Mastering BGP Best Path Selection

![BGP Best Path Selection: follow the rainbow to the pot of gold](images/bgp-best-path-cover.png)

> Study notes by [Nazirul Roslan](https://www.linkedin.com/in/nazirulroslan)

---

## 1. The Quest for the Best Path

When a BGP router has several paths to the same prefix, it runs a deterministic algorithm to pick a **single best path**. That path is installed in the routing table (if its administrative distance wins) and advertised to peers.

- **Unlike IGPs:** OSPF or EIGRP can install several equal-cost paths by default. BGP picks one best path (unless multipath is configured).
- **Sanity checks first:**
  - Is the **next hop** reachable (via an IGP, static, or connected route)?
  - Does the **AS path** contain the router's own AS? If so, the route is rejected (loop prevention).
- **New vs. existing prefix:** a brand-new prefix is simply marked best. If paths already exist, the algorithm runs.

---

## 2. The Decision Process (Cisco IOS)

Paths are compared step by step. **The first step where the paths differ decides the winner.**

A popular mnemonic for the core steps is **"N WLLA OMNI"**:

| # | Letter | Step | Prefer |
|---|---|---|---|
| 0 | **N** | Next hop | Reachable next hop (a prerequisite) |
| 1 | **W** | Weight | **Highest** (Cisco only, local; default 0, locally originated 32768) |
| 2 | **L** | Local Preference | **Highest** (shared across the AS via iBGP; default 100) |
| 3 | **L** | Locally originated | Paths from `network`, `redistribute` or `aggregate-address` on this router |
| 4 | **A** | AS path length | **Shortest** (AS_SET counts as 1, confederation segments don't count) |
| 5 | **O** | Origin code | **Lowest**: `i` (IGP) < `e` (EGP) < `?` (incomplete) |
| 6 | **M** | MED | **Lowest** (by default only compared between paths from the same neighbor AS) |
| 7 | **N** | Neighbor type | **eBGP** over iBGP |
| 8 | **I** | IGP metric to next hop | **Lowest** IGP cost to reach the BGP next hop |

> Newer IOS releases insert **AIGP** (lowest accumulated IGP metric, if the attribute is present) between locally originated and AS path length. It rarely appears outside designs that enable it.

### A note on the cover mnemonic "N WILLA ON ME"

The cover uses "N WILLA ON ME". It's a fine memory hook, but its letters don't line up one-to-one with the steps (there is no step for "I" between Weight and Local Preference). Use the table above as the reference for the actual order.

---

## 3. Further Tie-Breakers

If paths are still equal after the steps above:

| # | Step | Prefer |
|---|---|---|
| 9 | Multipath check | If `maximum-paths` is configured, install equal paths for load sharing (one is still marked best) |
| 10 | Oldest external path | The **oldest** eBGP path, for stability (skipped with `bgp bestpath compare-routerid`) |
| 11 | Router ID | **Lowest** BGP router ID of the advertising peer (or the ORIGINATOR_ID if the route was reflected) |
| 12 | Cluster list length | **Shortest** CLUSTER_LIST (route reflector designs) |
| 13 | Neighbor address | **Lowest** peer IP address |

---

## 4. Unique Scenarios & Considerations

- **Shortest AS path isn't always fastest:** BGP counts ASes, not physical hops or link speeds inside each AS.
- **MED comparison:** MEDs are only compared between paths from the **same neighboring AS**, unless `bgp always-compare-med` is configured.
- **eBGP over iBGP comes late:** Weight, Local Preference and AS path length are checked first, so an iBGP-learned path can easily beat an eBGP one.
- **Additional paths:** by default BGP advertises only its best path. The `additional-paths` feature advertises more than one, which helps convergence and load sharing but complicates "best path" reasoning.
- **`bgp bestpath compare-routerid`:** skips the "oldest external path" step, so eBGP ties are broken by router ID instead of age. This makes the result deterministic.

---

## 5. Troubleshooting Toolkit

```
show ip bgp
```
The BGP table. `>` marks the best path; weight, local preference, AS path and origin are visible.

```
show ip bgp <PREFIX>
```
Every path to one prefix with all attributes. The key command for understanding **why** one path won.

```
show ip bgp summary
```
Overview of neighbors and their states.

```
show ip bgp neighbors <IP> {advertised-routes | received-routes | routes}
```
Routes sent to, received from, or accepted from a neighbor. Verifies policy.

```
debug ip bgp updates [neighbor-ip] [in|out]
```
Traces update processing. **Use with extreme caution in production**, it can be very CPU intensive.

```
clear ip bgp * soft [in|out]
```
Re-applies policy without tearing down sessions.
