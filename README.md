# CML Network Labs

Cisco campus network labs built in Cisco Modeling Labs (CML) for build, verification, and break/fix practice. Lab work only, not production.

## Test-Lab-01: Inter-VLAN routing with a routed edge

```
[Alpine-10]─┐
            SW2 ══trunk══ SW1 ── R1 ── Lo0 8.8.8.8 ("internet")
[Alpine-20]─┘  (access)   (L3 core)
```

- VLANs 10/20, 802.1Q trunk, SVI-based inter-VLAN routing
- Routed uplink and static routing to the edge router
- SSH-only management access
- Verified layer by layer with CDP, VLAN/trunk, routing, and ping checks

**Break/fix:** 5 scenarios (pruned trunk VLAN, wrong access VLAN, SVI down, native VLAN mismatch, missing default route). See [`breakfix-log.md`](test-lab-01/breakfix-log.md).

**Run it:** import `test-lab-01/Test-Lab-01.yaml` into CML (Free tier works) and start the lab.

## Next

- Multi-site campus lab: EtherChannel, STP root design, DHCP, edge ACLs
- Python/Netmiko automation against the lab

*AI was used as a study aid; all builds and results are from my own lab runs.*
