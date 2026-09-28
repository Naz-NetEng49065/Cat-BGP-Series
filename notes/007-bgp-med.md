# Mastering BGP MED (Multi-Exit Discriminator)

![BGP MED: the MED value was too high!](images/bgp-med-cover.png)

> Study notes by [Nazirul Roslan](https://www.linkedin.com/in/nazirulroslan)

---

## 1. What is BGP MED?

The **Multi-Exit Discriminator (MED)**, often just called the BGP **metric**, is an **optional non-transitive** attribute. It lets you **suggest** to a neighboring AS which of your links it should use to send traffic **into** your AS, when you have several connections to that same neighbor.

```
            Your AS (65001)                    Neighbor AS (65002)
   Link A  ── advertises MED 100 ──►  prefers Link A (lower MED)
   Link B  ── advertises MED 200 ──►  uses Link B as backup
```

---

## 2. Key Characteristics

| Property | Detail |
|---|---|
| Type | **Optional non-transitive**: used by the neighbor AS (and shared inside it via iBGP), but **not** passed on to other ASes |
| Preference | **Lower is better** (the opposite of Weight and Local Preference) |
| Range | 32-bit: `0` – `4,294,967,295` |
| How to set | `set metric <value>` in a route map, usually applied outbound |
| Where you see it | The **Metric** column in `show ip bgp`, and the metric in the routing table |

### Default MED values (Cisco IOS)

- **`network` / `redistribute`:** the MED is copied from the IGP metric of the route in the routing table (so connected and static routes get `0`).
- **`aggregate-address`:** no MED is set.
- **Route received with no MED:** treated as `0` (the best possible value), unless you configure `bgp bestpath med missing-as-worst`, which treats it as the worst value.

---

## 3. Role in Best Path Selection

MED is checked **late**: after Weight, Local Preference, locally originated, AS path length and Origin. Only if all of those tie does MED matter, and then the **lowest MED wins**.

---

## 4. MED Comparison Rules (the tricky part)

### 1. Default: same neighboring AS only

By default Cisco only compares MEDs between paths from the **same neighboring AS** (the first AS in the AS path). MEDs from different ASes are ignored and the next tie-breaker decides.

### 2. `bgp deterministic-med`

Without it, IOS compares paths in the order they arrived (newest first), so the result can depend on arrival order. `bgp deterministic-med` first groups paths by neighboring AS and compares MEDs inside each group, making the result consistent.

> It is **disabled by default** on classic Cisco IOS / IOS-XE, so it must be configured. Cisco recommends enabling it on every router in the AS.

### 3. `bgp always-compare-med`

Compares MEDs from **any** eBGP neighbor, even in **different** ASes.

> ⚠️ Use with care. Every AS sets MED by its own rules, so comparing them across ASes can lead to odd routing. If you use it, enable it consistently on all routers in your AS.

| Command | MEDs compared between... | Default |
|---|---|---|
| *(none)* | Paths from the same neighbor AS, in arrival order | ✅ |
| `bgp deterministic-med` | Paths from the same neighbor AS, grouped, order-independent | Off |
| `bgp always-compare-med` | All paths, regardless of neighbor AS | Off |

---

## 5. Configuration Example

**Goal:** advertise `172.16.0.0/16` to neighbor `10.1.1.2` with MED 200, so this link is less preferred than another link to the same AS that advertises MED 100 (or 0).

```
ip prefix-list MY_PREFIXES permit 172.16.0.0/16
!
route-map SET_MED_OUT permit 10
 match ip address prefix-list MY_PREFIXES
 set metric 200             ! Lower is better, so 200 loses to 100
route-map SET_MED_OUT permit 20
 ! Other routes are advertised with their default MED
!
router bgp 65001
 neighbor 10.1.1.2 remote-as 65002
 neighbor 10.1.1.2 route-map SET_MED_OUT out
```

---

## 6. Common Use Cases

- **Influencing inbound traffic from one neighbor AS:** advertise a lower MED on the preferred link and a higher MED on backup links.
- **Alternative to AS-path prepending:** useful when the neighbor won't honor prepending, or when you only want to affect one neighbor.
- **Carrying IGP metrics (historical):** MED was designed to carry internal metrics to the neighbor. Today it's usually set to fixed policy values instead.

---

## 7. Best Practices & Key Considerations

- **Higher MED = less preferred.** Set a higher MED on the paths you want the neighbor to avoid.
- **It's only a suggestion.** The neighbor can override it, for example with Local Preference. Agree on the policy with them.
- **Enable `bgp deterministic-med`** on all routers in your AS for consistent results.
- **Use `bgp always-compare-med` sparingly**, and consistently if you do.
- **Remember non-transitive:** MED only reaches the directly connected neighbor AS.
- **Test and document:** MED changes can shift a lot of traffic. Lab-test, monitor, and record the values you use.

---

## 8. Troubleshooting Toolkit

```
show ip bgp
```
The **Metric** column shows each path's MED. Lower wins when MED is the deciding factor.

```
show ip bgp <NETWORK_IP> [mask <SUBNET_MASK>]
```
Example: `show ip bgp 172.16.10.0`. All paths to the prefix with their MED, and which one is best.

```
show ip route <NETWORK_IP>
```
For an installed BGP route, the metric in brackets is the MED, e.g. `[20/150]` = administrative distance 20, MED 150.

```
show ip bgp neighbors <NEIGHBOR_IP> advertised-routes
```
On the sending router: confirms your `set metric` is applied to outbound advertisements.

```
debug ip bgp updates [neighbor <IP>] [in|out]
```
Shows MED in sent/received updates. Very verbose; use carefully in production.

```
clear ip bgp <NEIGHBOR_IP> soft out
```
Re-sends your advertisements after changing an outbound MED route map, without resetting the session.

---

## 9. Glossary

| Term | Meaning |
|---|---|
| MED | Optional non-transitive attribute suggesting to a neighbor AS which entry point to use. Lower is better |
| Metric | In BGP output, usually the MED |
| Non-transitive | Not passed on beyond the neighboring AS |
| `set metric <value>` | Route-map command that sets MED |
| `bgp deterministic-med` | Groups paths by neighbor AS before comparing MED, removing arrival-order effects |
| `bgp always-compare-med` | Compares MED even between paths from different neighbor ASes |
| `bgp bestpath med missing-as-worst` | Treats a missing MED as the worst value instead of 0 |
