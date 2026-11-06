# 09 What to check when traffic fails

Troubleshoot in the same order that a packet travels. This makes it possible to distinguish a guest problem, a logical-config problem and a physical transport problem.

| **Question**                             | **Useful verification**                                   |
|------------------------------------------|-----------------------------------------------------------|
| Does the VM have the right vNIC and IP?  | virsh domiflist \<vm\>; ip addr and ip route inside guest |
| Is its tap on OVS and mapped?            | ovs-vsctl show; ovs-vsctl list Interface                  |
| Does the logical network exist?          | ovn-nbctl show; ovn-nbctl lsp-get-up \<port\>             |
| Where is the port bound?                 | ovn-sbctl show; inspect Port_Binding                      |
| Does OVN route as expected?              | ovn-trace (logical packet simulation)                     |
| Can the physical hosts reach each other? | ping / route to chassis IPs; underlay MTU                 |
| Are encapsulated packets visible?        | tcpdump -ni \<nic\> udp port 6081                         |
| Can the physical VLAN be reached?        | bridge mapping, port tags, switch trunk, tcpdump          |
| Is the gateway on the right chassis?     | ovn-nbctl lrp-get-gateway-chassis \<lrp\>                 |

ovn-trace tests the **logical** pipeline; it does not prove that a physical uplink, VLAN trunk, MTU or tunnel endpoint is functioning. Conversely, physical pings between KVM hosts do not prove that a VM port is bound to the correct logical switch. [2]

> **Diagnostic Habit:** Always identify the flow as local L2, cross-host L2, east–west L3, localnet/VLAN, or north–south NAT before following the packet.
