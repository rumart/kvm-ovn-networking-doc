# 03 How a VM joins a logical switch

A KVM guest has a virtio NIC. Libvirt commonly creates a tap device (vnetN) and attaches it to the OVS integration bridge. OVN needs to know which **logical switch port** that tap represents. [1, 7, 8]

For example, VM-A has MAC 02:ac:10:00:00:11, IP 10.10.10.11, and is attached to logical switch ls-app.

```bash
# Desired logical network (on an OVN admin host)
ovn-nbctl ls-add ls-app
ovn-nbctl lsp-add ls-app vm-a-port
ovn-nbctl lsp-set-addresses vm-a-port \
  "02:ac:10:00:00:11 10.10.10.11"
```

On the KVM host, the OVS Interface representing the VM must have external_ids:iface-id=vm-a-port. An integration layer (libvirt hook or orchestrator) normally ensures that the guest interface and the OVN port have matching IDs.

```bash
# Conceptual manual binding when vnet0 already exists
ovs-vsctl add-port br-int vnet0 -- \
  set Interface vnet0 external_ids:iface-id=vm-a-port
```

If libvirt itself attaches the VM to OVS, do not add the same tap a second time. Libvirt’s Open vSwitch virtualport uses an interfaceid UUID; make the OVN logical port name that same UUID, or implement a consistent mapping in your integration code. [7, 8]

> **Important:** Setting an IP in `lsp-set-addresses` records the port’s known addresses (not its port-security policy); it does not configure the guest operating system. Use static IP, guest DHCP or OVN DHCP options separately.
