# 12 Glossary

| **Term**        | **Meaning in this guide**                                  |
|-----------------|------------------------------------------------------------|
| Underlay        | The real IP/VLAN fabric between KVM hosts.                 |
| Overlay         | The logical Ethernet network transported in tunnels.       |
| br-int          | The OVS integration bridge on a chassis.                   |
| VIF / tap       | The host-side connection for a VM virtual NIC.             |
| Chassis         | An OVN transport host, compute or gateway.                 |
| Port binding    | OVN’s record of which chassis hosts a logical port.        |
| localnet        | An OVN port connecting logical L2 to existing physical L2. |
| Gateway chassis | Selected host for physical-facing gateway functions.       |
| SNAT / DNAT     | Source / destination IP address translation.               |
