# BGP Study Notes

Illustrated study notes on BGP path attributes, policy tools and design, based on Cisco IOS.

> By [Nazirul Roslan](https://www.linkedin.com/in/nazirulroslan)

| # | Topic | What it covers |
|---|---|---|
| 001 | [AS-Path](001-bgp-as-path.md) | Representations, loop prevention, prepending, `local-as`, `allowas-in` |
| 002 | [Weight](002-bgp-weight.md) | Cisco-only local preference knob, defaults, per-neighbor vs. route-map |
| 003 | [Route Maps](003-route-maps.md) | Sequences, match/set, the permit/deny puzzle, implicit deny |
| 004 | [Prefix Lists](004-prefix-lists.md) | `ge` / `le` logic and eight common patterns |
| 005 | [Best Path Selection](005-bgp-best-path-selection.md) | The full Cisco decision process and tie-breakers |
| 006 | [Origin Code](006-bgp-origin-code.md) | `i` vs. `e` vs. `?` and how each is set |
| 007 | [MED](007-bgp-med.md) | Influencing inbound traffic, `deterministic-med`, `always-compare-med` |
| 008 | [Enterprise Design](008-bgp-enterprise-design.md) | iBGP vs. eBGP internally, `remove-private-as` options |
| 009 | [Communities](009-bgp-communities.md) | Tagging, well-known communities, graceful shutdown |
| 010 | [Filtering Techniques](010-bgp-filtering-techniques.md) | ACLs vs. prefix lists, distribute-lists, route maps |
