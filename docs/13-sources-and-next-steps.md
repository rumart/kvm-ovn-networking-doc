# 13 Sources and next steps

The explanations are based on OVN’s own architecture and database documentation, with libvirt and OVS references for the VM attachment. All links below go to official project documentation.

## Primary official documentation

[[1] OVN Architecture](https://www.ovn.org/support/dist-docs/ovn-architecture.7.html)

[[2] OVN Sandbox Tutorial](https://docs.ovn.org/en/latest/tutorials/ovn-sandbox.html)

[[3] OVN Gateway High Availability](https://docs.ovn.org/en/latest/topics/high-availability.html)

[[4] OVN Northbound schema](https://www.ovn.org/support/dist-docs/ovn-nb.5.html)

[[5] OVN Southbound schema](https://www.ovn.org/support/dist-docs/ovn-sb.5.html)

[[6] OpenStack Networking introduction (concepts only)](https://docs.openstack.org/neutron/2026.2/admin/intro-os-networking.html)

[[7] libvirt Domain XML](https://libvirt.org/formatdomain.html)

[[8] Open vSwitch with libvirt](https://docs.openvswitch.org/en/latest/howto/libvirt/)

[[9] OVN Dynamic Routing architecture](https://docs.ovn.org/en/latest/topics/dynamic-routing/architecture.html)

The OpenStack Networking introduction is used only for general network, port, provider-network and routing terminology. This guide does not assume Neutron, Nova, ML2 or any other OpenStack component.

## Three useful lab exercises

- Move VM-B from kvm-b to kvm-a. Confirm that its logical port changes chassis while its IP stays the same.
- Add a second logical switch and route between two VM subnets. Use ovn-trace to explain each hop.
- Attach an external localnet and configure shared SNAT. Check that return traffic reaches the active gateway chassis.

---

*End of guide — keep the logical, physical and control-plane pictures separate.*
