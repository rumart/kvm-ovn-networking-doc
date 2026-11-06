# 06 How physical VLANs connect

A VM sometimes needs to be on a real Ethernet VLAN—for example, to reach a NAS, legacy VMs or bare-metal systems without routing. OVN uses a **localnet logical switch port** to join a logical switch to an existing physical L2 network. [1, 4, 5]

## Think “bridge mapping”, not “tunnel”

A host has an OVS provider bridge (br-provider) with a physical NIC or bond attached. The host advertises that physnet maps to that bridge. OVN then links the relevant logical switch’s localnet port to the mapped provider bridge (via OVS patch connectivity).

```bash
# Chassis-local mapping (illustrative)
ovs-vsctl set Open_vSwitch . \
  external_ids:ovn-bridge-mappings=physnet:br-provider
# OVN logical connection to that mapped network
ovn-nbctl ls-add ls-vlan220
ovn-nbctl lsp-add ls-vlan220 ln-vlan220
ovn-nbctl lsp-set-type ln-vlan220 localnet
ovn-nbctl lsp-set-addresses ln-vlan220 unknown
ovn-nbctl lsp-set-options ln-vlan220 network_name=physnet
ovn-nbctl set Logical_Switch_Port ln-vlan220 tag_request=220
```

Set the appropriate VLAN tag (for example 220) on the localnet port, and make the physical switch carry VLAN 220 to the host. OVN/OVS handle the tagging at the boundary; the guest can see an untagged vNIC. The physical VLAN becomes part of the datapath.

> **Avoid An Easy Mistake:** `br-provider` does not need the SNAT address as a Linux interface IP. A router gateway address and its NAT address are OVN logical configuration; the OVS provider bridge is primarily a Layer-2 attachment.
