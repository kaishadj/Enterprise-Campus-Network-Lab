# Traffic policy and management access

## Inter-VLAN and guest filtering

| Access list | Intended attachment | Policy |
| --- | --- | --- |
| `ACCOUNTING-IN` | Inbound on `Vlan20` on both distribution switches | Deny Accounting to Sales and permit other IPv4 traffic. |
| `SALES-IN` | Inbound on `Vlan30` on both distribution switches | Deny Sales to Accounting and permit other IPv4 traffic. |
| `VLAN50-HSRP-IN` | Inbound on `Vlan50` on both distribution switches | Allow HSRPv1 hellos from the two distribution switches, DHCP traffic for guests, and ICMP echo and HTTP to `203.0.113.10`. Deny all other IPv4 traffic. |

Both distribution switches have the ACL definitions. The guest filter includes HSRPv1 UDP/1985 to `224.0.0.2` so the two switches can agree on which one owns the guest gateway. The DHCP exceptions cover the initial broadcast and a renewal addressed to `192.168.40.10`.

The isolation test verified that DSW2 preserved access to WEB-EXT and blocked an ICMP request to IT-1. The earlier running-config snapshots show the named ACLs and omit the `ip access-group` commands under the SVIs. After reopening the `.pkt`, check the policy with the [traffic tests](../verification/README.md#filtering-after-reopening). The ICMP and deny counters can confirm which rules handled the test packets. [Recorded guest pings](../screenshots/guest-1-filtering-dsw1-isolated.png) · [ACL counters](../screenshots/dsw2-guest-acl-counters.png)

If filtering stops after a reload, reapply the existing lists on **both DSW1 and DSW2**.

```text
configure terminal
interface vlan 20
 ip access-group ACCOUNTING-IN in
exit
interface vlan 30
 ip access-group SALES-IN in
exit
interface vlan 50
 ip access-group VLAN50-HSRP-IN in
end
```

From GUEST-1, check that `ping 203.0.113.10` succeeds and `ping 192.168.10.11` fails. Verify from IT-1 that `192.168.10.11` is its current address. On the active distribution switch, run `show access-lists VLAN50-HSRP-IN` to compare counters before and after the tests.

## Switch management

The three access switches have SSH version 2, local login and management IPs on VLAN 99. ASW-IT additionally applies standard ACL 10 to its VTY lines and permits only the static IT-1 IP `192.168.10.11`. The recorded incident confirmed that an IT-1 DHCP lease at `192.168.10.51` could ping ASW-IT but was refused for SSH. Restoring the static address let the administrator connect again. The ASW-ACCOUNTING and ASW-SALES exports configure SSH and local login without that VTY source restriction.

## Simulated external access

EDGE-R1 uses NAT overload on outside interface `Gi0/1`. `Gi0/0` and `Gi0/2` both face the campus and are configured as NAT inside. Standard ACL 1 selects the IT, Accounting, Sales, and Guest networks for translation. Both campus uplinks can therefore use `198.51.100.1` as the translated source address when accessing the external test server. [ICMP translation evidence](../screenshots/edge-r1-pat-it-1-web-ext.png)
