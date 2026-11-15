# Troubleshooting

## Accounting trunk used different native VLANs

DSW1 `Fa0/2` reported `NATIVE_VLAN_MISMATCH`. Its native VLAN was 1, while ASW-ACCOUNTING `Gi0/1` used 999. VLAN 20 also needed to be created on DSW1. I brought the two trunk configurations into agreement and connected a temporary VLAN 20 test PC to DSW1. The ping across the trunk succeeded with four replies. [Ping evidence](../screenshots/accounting-vlan20-trunk-ping.png)

## ASW-IT elected itself as the VLAN 10 root

CDP found the neighbor and the gateway replied to pings, but ASW-IT still did not select the link to DSW1 as its root port. I moved the DSW1 end of the trunk from `Fa0/1` to `Fa0/12` and kept its trunk settings. With Rapid PVST+ restored, ASW-IT recognized DSW1 as root and put `Gi0/1` in `Root FWD`. The exact reason for the behavior on `Fa0/1` remains unconfirmed. [Rapid PVST+ output](../screenshots/asw-it-rapid-pvst-vlan10-root-fa0-12.png)

## LACP suspended a member because its VLAN list differed

`%EC-5-CANNOT_BUNDLE2` reported a VLAN mask mismatch between `Fa0/24` and `Po1`. I matched the allowed VLANs, native VLAN, and trunk mode on both member ports and the port channel. After rejoining the LACP group, both ports appeared as `(P)`. With `Fa0/24` disabled, `Po1(SU)` stayed up on `Fa0/23`.

## The old LACP bundle failed to recover after DSW1 isolation

After I restored the links, ports on the old group appeared as `(I)` or stood down, depending on which switch and point in the test I checked. Recreating the group did not recover it. I moved the inter-switch connection to `Fa0/21` and `Fa0/22` and configured LACP group 2 on both switches. Both links appeared as `(P)` under `Po2(SU)`. DSW2 used `Po2` as its VLAN 10 root port. The reason the original bundle failed to recover remains unconfirmed. [DSW1 output](../screenshots/dsw1-po2-lacp-bundle.png) · [DSW2 output](../screenshots/dsw2-po2-lacp-stp-recovery.png)

## SSH was refused although ping succeeded

ASW-IT replied on its management IP and `show ip ssh` reported SSH version 2. IT-1 had received `192.168.10.51` through DHCP, while the VTY access list permitted only `192.168.10.11`. I restored the static address outside the DHCP pool and verified that an SSH session could open. Ping checked IP reachability. SSH also required the source IP to pass the VTY access list.
