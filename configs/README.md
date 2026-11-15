# Device configurations

These are IOS configuration snapshots. The DSW1 and DSW2 files include the SVI ACL commands entered during the live filtering tests, which the raw exports omitted. Local credential lines were removed. To rebuild SSH locally, create a lab-only `netadmin` account and generate RSA keys on each access switch, then verify the VTY settings.

| Layer | Configuration exports |
| --- | --- |
| Access | [ASW-IT](ASW-IT.cfg), [ASW-ACCOUNTING](ASW-ACCOUNTING.cfg), [ASW-SALES](ASW-SALES.cfg) |
| Distribution | [DSW1](DSW1.cfg), [DSW2](DSW2.cfg) |
| Edge and simulated ISP | [EDGE-R1](EDGE-R1.cfg), [ISP-R1](ISP-R1.cfg) |

Packet Tracer device settings also include the access VLAN database and end-host services. Use the [Packet Tracer project](../packet-tracer/Enterprise-Campus-Network.pkt) to inspect those settings live and the [network design](../documentation/network-design.md) to check port assignments and server settings.
