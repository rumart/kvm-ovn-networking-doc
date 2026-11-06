# 02 The OVN components

## Control plane: describes and distributes intent

- **Northbound DB (NB):** desired logical switches, ports, routers, ACLs and NAT rules. An operator or automation writes here.
- **ovn-northd:** translates the logical configuration into lower-level logical flows and port-binding information.
- **Southbound DB (SB):** the shared view of logical flows, chassis identities, tunnel endpoints and where VM ports currently live.
- **ovn-controller (each host):** reads SB and programs the local OVS dataplane; reports the chassis and bound VIFs.

## Dataplane: actually forwards packets

**Open vSwitch (OVS)** is the software switch on each host, typically with an OVN integration bridge named br-int. It handles locally attached tap ports, applies OVN-programmed flows and sends packets through tunnels or physical uplinks. The VM never communicates with the NB or SB database itself. [1]

**Chassis** means an OVN transport node: usually a KVM compute host or a gateway host. The controllers and databases can run on other machines. A central controller is not in the normal per-packet path.

> **Availability Implication:** If the control plane becomes temporarily unavailable, established OVS forwarding normally continues with the programmed state, but new bindings and network changes may not propagate.
