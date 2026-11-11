# Break/Fix Log: Test-Lab-01

Lab environment: Cisco Modeling Labs (CML), IOL / IOL-L2 nodes, Alpine Linux hosts.
This is **lab work**, not production.

## Topology

```
[Alpine-10] ─eth0── e0/2 ┐
                          SW2 ── e0/1 ══ trunk ══ e0/1 ── SW1 ── e0/0 ─── e0/0 ── R1 ── Lo0 8.8.8.8
[Alpine-20] ─eth0── e0/3 ┘   (access)                  (L3 core)   10.0.0.1/30   10.0.0.2/30
```

| Item | Value |
|---|---|
| VLAN 10 (USERS) | 192.168.10.0/24, gateway 192.168.10.1 (SW1 SVI) |
| VLAN 20 (CCTV) | 192.168.20.0/24, gateway 192.168.20.1 (SW1 SVI) |
| SW1 ↔ R1 | 10.0.0.0/30 (SW1 .1, R1 .2) |
| "Internet" | R1 Loopback0 8.8.8.8/32 |
| Routing | SW1 default route → 10.0.0.2; R1 static 192.168.0.0/16 → 10.0.0.1 |

---

## Baseline: build and verification

| # | Check | Command | Result |
|---|---|---|---|
| 0 | Cabling matches the diagram | `show cdp neighbors` (SW1, SW2, R1) | ✅ |
| 1 | Access ports in the correct VLANs | `show vlan brief` (SW2) | ✅ |
| 2 | Trunk up, VLANs 10/20 forwarding | `show interfaces trunk` (SW2) | ✅ |
| 3 | SVIs and uplink up/up | `show ip interface brief` (SW1) | ✅ |
| 4 | Host reaches its gateway | `ping -c 3 192.168.10.1` (Alpine-10) | ✅ |
| 5 | Inter-VLAN routing | `ping -c 3 192.168.20.10` (Alpine-10) | ✅ |
| 6 | Routing table correct | `show ip route` (SW1) | ✅ |
| 7 | Routed path to the edge and back | `ping -c 3 8.8.8.8` (Alpine-10) | ✅ |
| 8 | MAC learning | `show mac address-table` (SW1) | ✅ |
| 9 | SSH on SW1 (local user, vty SSH only, RSA 2048) | `ssh -l admin 10.0.0.1` from R1 | ✅ |

**Config notes**
- On IOL-L2, `switchport trunk encapsulation dot1q` must come before `switchport mode trunk`.
- `ip routing` is what lets SW1 route between VLANs.
- R1 needs a return route to 192.168.0.0/16 so replies can get back to the VLANs.
- Verify bottom-up (L1 → L2 → L3) so a failure points at one layer.

---

## Break/fix scenarios

Format: **Break → Symptom → Diagnosis → Root cause → Fix → Verified**

### BF-01: VLAN missing from a trunk's allowed list
- **Status:** ✅ Done
- **Break (SW2):** `interface e0/1` → `switchport trunk allowed vlan 10`
- **Symptom:** Alpine-20 couldn't ping its gateway (192.168.20.1). Alpine-10 could still ping its gateway.
- **Diagnosis:** `show interfaces trunk` on SW2 showed only VLAN 10 allowed on Et0/1, while SW1's side still allowed all VLANs.
- **Root cause:** VLAN 20 was pruned from SW2's side of the trunk, so VLAN 20 frames never reached SW1.
- **Fix:** `interface e0/1` → `switchport trunk allowed vlan 10,20`
- **Verified:** `show interfaces trunk` listed 10,20; Alpine-20 pinged 192.168.20.1.

### BF-02: Access port in the wrong VLAN (camera on the wrong VLAN)
- **Status:** ✅ Done
- **Break (SW2):** `interface e0/3` → `switchport access vlan 10`
- **Symptom:** Alpine-20 (192.168.20.10, gateway .20.1) couldn't reach its gateway or anything else, even though the link was up.
- **Diagnosis:** `show vlan brief` showed Et0/3 under VLAN 10; `show mac address-table interface e0/3` showed Alpine-20's MAC learned in VLAN 10.
- **Root cause:** The host's subnet (VLAN 20) didn't match the port's VLAN (10), so its ARP for 192.168.20.1 never reached the VLAN 20 SVI.
- **Fix:** `interface e0/3` → `switchport access vlan 20`
- **Verified:** `show vlan brief` showed Et0/3 in VLAN 20; Alpine-20 pinged its gateway.
- **Work tie-in:** Same failure pattern as CCTV cameras showing offline on an NVR because of wrong VLAN assignment.

### BF-03: Gateway SVI down
- **Status:** ✅ Done
- **Break (SW1):** `interface vlan 20` → `shutdown`
- **Symptom:** Alpine-20 couldn't reach its gateway; Alpine-10 couldn't reach 192.168.20.10. Layer 2 still looked healthy.
- **Diagnosis:** `show ip interface brief` showed Vlan20 *administratively down*; `show ip route` no longer listed 192.168.20.0/24 as connected.
- **Root cause:** The VLAN 20 gateway was shut down, so VLAN 20 had no path off its subnet.
- **Fix:** `interface vlan 20` → `no shutdown`
- **Verified:** Vlan20 up/up; inter-VLAN ping worked again.

### BF-04: Native VLAN mismatch
- **Status:** ✅ Done
- **Break (SW2):** `interface e0/1` → `switchport trunk native vlan 99` (SW1 left on native VLAN 1)
- **Symptom:** Pings kept working: Alpine-20 could still ping its gateway (192.168.20.1) and Alpine-10 (192.168.10.10). The only sign of the problem was CDP native VLAN mismatch messages in both SW1's and SW2's logs.
- **Diagnosis:** `%CDP-4-NATIVE_VLAN_MISMATCH` in the logs; `show interfaces trunk` on both switches showed different native VLANs.
- **Root cause:** The two ends of the trunk disagreed on which VLAN is untagged. VLANs 10 and 20 are tagged, so host traffic wasn't affected.
- **Fix:** `interface e0/1` → `no switchport trunk native vlan 99`
- **Verified:** Native VLAN matched on both ends; no new mismatch messages.
- **Lesson:** Not every misconfiguration breaks pings. Check the logs, not just ping results.

### BF-05: Missing default route
- **Status:** ✅ Done
- **Break (SW1):** `no ip route 0.0.0.0 0.0.0.0 10.0.0.2`
- **Symptom:** Inter-VLAN routing still worked, but neither Alpine could ping the "internet" (8.8.8.8).
- **Diagnosis:** `show ip route` showed *Gateway of last resort is not set* and no `S*` entry.
- **Root cause:** SW1 had no route for destinations outside its connected networks.
- **Fix:** `ip route 0.0.0.0 0.0.0.0 10.0.0.2`
- **Verified:** `S* 0.0.0.0/0 via 10.0.0.2` was back; 8.8.8.8 replied.
