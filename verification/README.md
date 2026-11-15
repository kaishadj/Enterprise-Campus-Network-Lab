# Verification

Use Cisco Packet Tracer 9.0.0 in Realtime mode. Run `enable` before the IOS checks. Save a separate copy of the lab for tests that disable interfaces.

## Baseline

| Device | Commands or actions | What to check |
| --- | --- | --- |
| DSW1 and DSW2 | `show etherchannel summary`, `show interfaces trunk` | `Po2(SU)` with `Fa0/21(P)` and `Fa0/22(P)`. Native VLAN 999 on each trunk. |
| DSW1 and DSW2 | `show standby brief` | DSW1 active and DSW2 standby in groups 10, 20, 30, 40, 50, and 99. |
| ASW-IT | `show spanning-tree vlan 10` | Root ID belongs to DSW1. `Gi0/1` is `Root FWD` and a working backup path can appear as `Altn BLK` on `Gi0/2`. |
| EDGE-R1 | `show ip ospf neighbor`, `show ip route ospf` | Routes to the campus subnets learned over the links to the distribution switches. |
| IT-1 | `ping 192.168.10.1`, `ping 203.0.113.10` | The shared gateway and simulated external server reply. |
| EDGE-R1 | `show ip nat translations` immediately after external pings | IT-1 `192.168.10.11` is translated to `198.51.100.1` for traffic to `203.0.113.10`. [Captured translation table](../screenshots/edge-r1-pat-it-1-web-ext.png) |

Selected command output is available as a [text transcript](cli-observations.txt).

## Filtering after reopening

Open the saved `.pkt` and wait for the campus to converge. Run the tests before applying any configuration commands.

| Source | Destination | Expected result |
| --- | --- | --- |
| GUEST-1 | `203.0.113.10` | WEB-EXT replies. |
| GUEST-1 | `192.168.10.11` | IT-1 receives no allowed traffic from GUEST-1. |
| ACCOUNTING-1 and SALES-1 | The other PC's current IPv4 address, from `ipconfig` | The ping is blocked in both directions. |

If a blocked destination responds, follow the [ACL attachment steps](../documentation/security.md#inter-vlan-and-guest-filtering) and repeat the checks. Compare the ACL counters on the active distribution switch before and after the pings.

## Failover and recovery tests

| Test | How to repeat | Recorded result |
| --- | --- | --- |
| IT uplink loss | Shut `ASW-IT Gi0/1`. Check `show spanning-tree vlan 10` and ping `192.168.10.1` from IT-1. Apply `no shutdown` to restore the link. | `Gi0/2` became `Root FWD` and IT-1 received 4/4 replies after convergence. This test used the original `Po1` bundle. [Before](../screenshots/asw-it-stp-redundant-uplink-baseline.png) · [After](../screenshots/asw-it-stp-failover-root-port.png) · [Ping](../screenshots/it-1-ping-via-backup-uplink.png) |
| VLAN 10 gateway loss | Shut `Vlan10` on DSW1. Check HSRP on DSW2, OSPF on EDGE-R1, and ping WEB-EXT from IT-1. Restore the SVI with `no shutdown`. | DSW2 went from `Standby` to `Active`, EDGE-R1 learned `192.168.10.0/24` via DSW2, and IT-1 received 4/4 replies after the change. [HSRP](../screenshots/dsw2-hsrp-vlan10-active.png) · [OSPF](../screenshots/edge-r1-it-ospf-via-dsw2.png) · [Ping](../screenshots/it-1-web-ext-ping-via-dsw2.png) |
| DSW1 isolation | Disable the connected links on DSW1. Use previously addressed clients. Check HSRP on DSW2, OSPF on EDGE-R1, and ping the external server. | DSW2 took over all six HSRP groups. EDGE-R1 showed routes to VLANs 10, 20, 30 and 50 via `10.255.0.6`. IT-1 got one timeout and then three replies from WEB-EXT. The route to the server VLAN was absent during isolation. [Topology](../screenshots/dsw1-isolated-topology.png) · [HSRP](../screenshots/dsw2-hsrp-dsw1-isolated.png) · [OSPF](../screenshots/edge-r1-ospf-dsw1-isolated.png) · [Ping](../screenshots/it-1-web-ext-dsw1-isolated.png) |
| Guest policy on the backup path | During isolation, from GUEST-1 ping `203.0.113.10` and `192.168.10.11`. On DSW2 check `show access-lists VLAN50-HSRP-IN`. | WEB-EXT replied 4/4. DSW2 `192.168.50.3` returned `Destination host unreachable` for the IT destination. The ACL counters are cumulative. [Pings](../screenshots/guest-1-filtering-dsw1-isolated.png) · [ACL](../screenshots/dsw2-guest-acl-counters.png) |
| Return to normal | Re-enable the connected links and compare `show etherchannel summary`, `show spanning-tree vlan 10`, and `show standby brief` to the baseline. | After the Po2 repair, both physical members were bundled on both 3560s. DSW2 used `Po2` as its VLAN 10 root port. DSW1 resumed the active role for all six groups. [DSW1 LACP](../screenshots/dsw1-po2-lacp-bundle.png) · [DSW2 LACP and STP](../screenshots/dsw2-po2-lacp-stp-recovery.png) · [HSRP](../screenshots/dsw1-hsrp-active-after-recovery.png) |

During a full DSW1 isolation, expect SRV-INFRA to be disconnected. The collected results show recovery after a failure. Convergence time and uninterrupted packet delivery were outside the measurements.

On the final cabling, the connected DSW1 interfaces for the full isolation test are `Fa0/2`, `Fa0/3`, `Fa0/10`, `Fa0/12`, `Fa0/21`, `Fa0/22`, and `Gi0/1`. Restore those same links after the test and confirm that both Po2 members have rejoined before saving the restored test file.
