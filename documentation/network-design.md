# Network design

## Devices and roles

| Function | Devices | Role |
| --- | --- | --- |
| Access | ASW-IT, ASW-ACCOUNTING, ASW-SALES · Cisco 2960-24TT | Connect PCs to their VLANs and carry tagged traffic to both distribution switches. |
| Distribution | DSW1, DSW2 · Cisco 3560-24PS | Route VLAN traffic through switch virtual interfaces (SVIs). Provide shared HSRP gateways and Rapid PVST+ root redundancy. |
| Edge and provider | EDGE-R1, ISP-R1 · Cisco 2911 | EDGE-R1 runs OSPF toward the campus and performs PAT. ISP-R1 routes to the simulated external network. |
| Services | SRV-INFRA and WEB-EXT · Server-PT | SRV-INFRA supplies DHCP, DNS, and intranet HTTP. WEB-EXT provides the external test destination. |

## VLANs and gateways

| VLAN | Name | Network | HSRP virtual IP | DSW1 IP | DSW2 IP |
| --- | --- | --- | --- | --- | --- |
| 10 | IT | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.2` | `192.168.10.3` |
| 20 | ACCOUNTING | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.2` | `192.168.20.3` |
| 30 | SALES | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.2` | `192.168.30.3` |
| 40 | SERVERS | `192.168.40.0/24` | `192.168.40.1` | `192.168.40.2` | `192.168.40.3` |
| 50 | GUEST | `192.168.50.0/24` | `192.168.50.1` | `192.168.50.2` | `192.168.50.3` |
| 99 | MANAGEMENT | `192.168.99.0/24` | `192.168.99.1` | `192.168.99.2` | `192.168.99.3` |
| 999 | NATIVE-UNUSED | — | — | — | — |

The `.1` addresses are the default gateways used by endpoints. HSRPv1 uses the VLAN ID as the group number. DSW1 has priority 110 and preempt enabled. DSW2 uses priority 100. The two switches can take turns answering for the same gateway IP, so clients retain their gateway setting during a failover.

Rapid PVST+ chooses a forwarding path per VLAN. The configured base priorities are 24576 on DSW1 and 28672 on DSW2. The primary and secondary uplinks are `Gi0/1` and `Gi0/2` on each access switch. On a redundant triangle, STP can keep one link in the alternate state until the active path fails.

## Trunk and port map

| Link | Port A | Port B | Allowed VLANs |
| --- | --- | --- | --- |
| DSW1 → ASW-IT | `Fa0/12` | `Gi0/1` | 10, 99, 999 |
| DSW2 → ASW-IT | `Fa0/1` | `Gi0/2` | 10, 99, 999 |
| DSW1 → ASW-ACCOUNTING | `Fa0/2` | `Gi0/1` | 20, 99, 999 |
| DSW2 → ASW-ACCOUNTING | `Fa0/2` | `Gi0/2` | 20, 99, 999 |
| DSW1 → ASW-SALES | `Fa0/3` | `Gi0/1` | 30, 50, 99, 999 |
| DSW2 → ASW-SALES | `Fa0/3` | `Gi0/2` | 30, 50, 99, 999 |
| DSW1 → DSW2 | `Po2` (`Fa0/21–22`) | `Po2` (`Fa0/21–22`) | 10, 20, 30, 40, 50, 99, 999 |

Every active trunk uses VLAN 999 as the native VLAN. VLAN 99 carries tagged management traffic. The old `Po1` member ports (`Fa0/23–24`) are shut down in the exported configs. Access-switch ports `Fa0/1` and `Fa0/2` serve their department PCs. `ASW-SALES Fa0/3` serves GUEST-1. `DSW1 Fa0/10` connects SRV-INFRA as an access port in VLAN 40.

## Routed links and OSPF

| Link | Interface and address A | Interface and address B | Subnet |
| --- | --- | --- | --- |
| EDGE-R1 → DSW1 | `Gi0/0` · `10.255.0.1` | `Gi0/1` · `10.255.0.2` | `10.255.0.0/30` |
| EDGE-R1 → DSW2 | `Gi0/2` · `10.255.0.5` | `Gi0/1` · `10.255.0.6` | `10.255.0.4/30` |
| EDGE-R1 → ISP-R1 | `Gi0/1` · `198.51.100.1` | `Gi0/0` · `198.51.100.2` | `198.51.100.0/30` |
| ISP-R1 → WEB-EXT | `Gi0/1` · `203.0.113.1` | `Fa0` · `203.0.113.10` | `203.0.113.0/24` |

The `/30` networks give each point-to-point link its own subnet with exactly two usable IPv4 addresses. OSPFv2 process 1 uses area 0. Each 3560 advertises the five campus networks in VLANs 10, 20, 30, 40, and 50 when their SVIs are up. VLAN 99 stays outside the OSPF network statements. EDGE-R1 originates the default route into OSPF and forwards external traffic to `198.51.100.2`.

## Static endpoints and services

| Endpoint | IPv4 address | Default gateway |
| --- | --- | --- |
| IT-1 | `192.168.10.11/24` | `192.168.10.1` |
| ASW-IT | `192.168.99.11/24` | `192.168.99.1` |
| ASW-ACCOUNTING | `192.168.99.12/24` | `192.168.99.1` |
| ASW-SALES | `192.168.99.13/24` | `192.168.99.1` |
| SRV-INFRA | `192.168.40.10/24` | `192.168.40.1` |
| WEB-EXT | `203.0.113.10/24` | `203.0.113.1` |

SRV-INFRA serves the client DHCP pools for VLANs 10, 20, 30, and 50. Pool allocation starts at `.50`. Client settings include the HSRP gateway for their subnet. On DSW1, the four client SVIs relay DHCP to `192.168.40.10`. DSW2 relays DHCP for VLAN 50. The internal DNS record for `intranet.atlas.test` points to SRV-INFRA. The guest test uses WEB-EXT by IP.

The [logical diagram](../topology/logical-topology.svg) gives the final cabling. The [Packet Tracer topology capture taken during DSW1 isolation](../screenshots/dsw1-isolated-topology.png) shows the earlier `Po1` wiring and the interfaces down in that test.
